---
title: Generating Cgo for libvirt
date: 2023-01-01
draft: true
tags:
    - go
    - libvirt
categories:
    - coding
keywords:
    - cgo
    - libvirt
---

# Preface

The [libvirt-go-module][] project provides Go bindings to use libvirt's library. It wraps libvirt's
C types and APIs into Go types and methods. The wrapping involves writing a bit of [Cgo code][] that
looks like the following:

{{< highlight c "linenos=table, hl_lines=6, linenostart=759" >}}
/* connect_wrapper.go from libvirt-go-module version v1.8009.0 */
virConnectPtr
virConnectOpenWrapper(const char *name,
                      virErrorPtr err)
{
    virConnectPtr ret = virConnectOpen(name);
    if (!ret) {
        virCopyLastError(err);
    }
    return ret;
}
{{< /highlight >}}

In this example, we are calling libvirt's [virConnectOpen()][]  to stablish connection with the
underlying hypervisor. The `virConnectOpenWrapper()` is a helper function, to be consumed by the
`NewConnect()` API, which returns a pointer of Go struct type `Connect` to the caller or sets the
error if unsuccessful. The actual code:

{{< highlight go "linenos=table, hl_lines=9, linenostart=328" >}}
/* connect.go from libvirt-go-module version v1.8009.0 */
func NewConnect(uri string) (*Connect, error) {
	var cUri *C.char
	if uri != "" {
		cUri = C.CString(uri)
		defer C.free(unsafe.Pointer(cUri))
	}
	var err C.virError
	ptr := C.virConnectOpenWrapper(cUri, &err)
	if ptr == nil {
		return nil, makeError(&err)
	}
	return &Connect{ptr: ptr}, nil
}
{{< /highlight >}}

# The problem

[Libvirt][] is a big project created in 2005, currently with over 1.1M lines of code[^1] with an
average of 3800 commits per year[^2]. New APIs are constantly being added to handle new features
which raises the need of `libvirt-go-module` to be patched constantly with Cgo glue code to wrap
around libvirt's new additions plus the actual Go code that will expose it to Go applications.

Reducing this work as much as possible would help with the maintenance of `libvirt-go-module` while
making it easier to keep go bindings up to date with libvirt.

Luckily, libvirt's API is [thoroughly documented][] in a format that we can take advantage of in
order to generate almost all of the Cgo code, leaving developers able to focus on how to expose
libvirt's new features.

# The input

Libvirt's `scripts/apibuild.py` parses the source code to gather all the metadata it can to build
its API documentation which is an XML format. The `<api>` section has two subsections, `<files>` and
`<symbols>`. The symbols part is what we are mostly interested in order to generate those Cgo
wrappers. A redacted example is below:

Should add the fact it is split by modules, -lxc, -qemu, -admin and that libvirt-go-module can be
built without support of lxc and qemu.

{{< highlight xml >}}
<?xml version="1.0" encoding="UTF-8"?>
<api name='libvirt'>
  <files>
    <! 24 file names in total >
    <file name='libvirt-domain-checkpoint'>
     <summary>APIs for management of domain checkpoints</summary>
     <description>Provides APIs for the management of domain checkpoints</description>
     <exports symbol='VIR_DOMAIN_CHECKPOINT_CREATE_QUIESCE' type='enum'/>
    </file>
  </files>
  <symbols>
    <! 285 macros in total >
    <macro name='VIR_CONNECT_IDENTITY_GROUP_NAME'
           file='libvirt-host'
           string='group-name'
           version='5.8.0'>
      <info><! redacted documentation ></info>
    </macro>
    <! 1025 enums in total >
    <enum name='VIR_CONNECT_BASELINE_CPU_EXPAND_FEATURES'
          file='libvirt-host'
          value='1'
          value_hex='0x1'
          value_bitshift='0'
          type='virConnectBaselineCPUFlags'
          version='1.1.2'
          info='show all features'/>
    <! 49 structs in total >
    <struct name='virConnect'
            file='libvirt-host'
            type='struct _virConnect'
            version='0.0.1'/>
    <! 218 typedefs in total >
    <typedef name='virConnectBaselineCPUFlags'
             file='libvirt-host'
             type='enum'
             version='1.1.2'>
      <info><! redacted documentation ></info>
    </typedef>
    <! 1 variable >
    <variable name='virConnectAuthPtrDefault'
              file='libvirt-host'
              type='virConnectAuthPtr'
              version='0.4.1'>
      <info><! redacted documentation ></info>
    </variable>
    <! 538 functions in total >
    <function name='virConnResetLastError'
              file='virterror'
              module='virerror'
              version='0.1.0'>
      <info><! redacted documentation ></info>
      <return type='void'/>
      <arg name='conn'
           type='virConnectPtr'
           info='pointer to the hypervisor connection'/>
    </function>
    <! 52 functype in total >
    <functype name='virConnectAuthCallbackPtr'
              file='libvirt-host'
              module='libvirt-host'
              version='0.4.1'>
      <info><! redacted documentation ></info>
      <return type='int'
              info='0 if all interactions were filled, or -1 upon error'/>
      <arg name='cred'
           type='virConnectCredentialPtr'
           info='list of virConnectCredential object to fetch from user'/>
      <arg name='ncred'
           type='unsigned int'
           info='size of cred list'/>
      <arg name='cbdata'
           type='void *'
           info='opaque data passed to virConnectOpenAuth'/>
    </functype>
  </symbols>
</api>

{{< /highlight >}}

# Adding version metadata

Should add the fact that version was not included prior to version X.Y.Z for most of the symbols.

Also, one of `libvirt-go-module`'s goals is to be able to compile and run in platforms supported by
libvirt itself.  This means we need to be aware of libvirt's version we at compile and running time.
At compile time, we might need libvirt symbols that are not included in older versions of its
headers; At run time, to avoid calling functions that are not yet implement, like `SetIdentity`
below:

{{< highlight go "linenos=table, linenostart=582" >}}
// See also https://libvirt.org/html/libvirt-libvirt-host.html#virConnectSetIdentity
func (c *Connect) SetIdentity(ident *ConnectIdentity, flags uint32) error {
    if C.LIBVIR_VERSION_NUMBER < 5008000 {
        return makeNotImplementedError("virConnectSetIdentity")
    }
    /* implements the function  */
{{< /highlight >}}

# The code generator

1. The reading-from-file to Go structs.
2. The templates for Cgo.
3. go generate

# Compiling without libvirt headers

We should add the dlopen dlsym work here:
- changes in the generator
- changes in the build process

# Conclusion


[libvirt-go-module]: https://gitlab.com/libvirt/libvirt-go-module/
[Cgo code]: https://go.dev/blog/cgo
[virConnectOpen()]: https://libvirt.org/html/libvirt-libvirt-host.html#virConnectOpen
[Libvirt]: https://libvirt.org/
[thoroughly documented]: https://libvirt.org/html/index.html

[^1]: Tokei
| Language         |    Files  |     Lines |     Code  |  Comments   |  Blanks |
| :--------------- | --------: | --------: | --------: | ----------: | ------: |
| Alex             |      9    |    8118   |     6710  |         0   |    1408 |
| Autoconf         |     86    |    8080   |     5190  |      1886   |    1004 |
| BASH             |      1    |      94   |       81  |         1   |      12 |
| C                |    635    |  658853   |   474652  |     69777   |  114424 |
| C Header         |    484    |  157396   |   106061  |     22425   |   28910 |
| C++              |      1    |      40   |       22  |         8   |      10 |
| CSS              |      5    |     890   |      757  |         6   |     127 |
| D                |      2    |     101   |       84  |         0   |      17 |
| Dockerfile       |     34    |    3673   |     3358  |       170   |     145 |
| JavaScript       |      1    |     141   |      116  |         1   |      24 |
| JSON             |    304    |   46250   |    46095  |         0   |     155 |
| Makefile         |      3    |    2057   |     1378  |       405   |     274 |
| Meson            |     74    |    9318   |     7767  |       429   |    1122 |
| Perl             |      4    |    3620   |     2915  |       283   |     422 |
| Python           |     41    |   13427   |    10000  |      1272   |    2155 |
| ReStructuredText |    159    |   59502   |    44465  |         0   |   15037 |
| Rust             |      1    |      28   |       22  |         0   |       6 |
| Shell            |     61    |    5871   |     4822  |       513   |     536 |
| SVG              |     18    |    7223   |     6915  |       301   |       7 |
| Plain Text       |     12    |     173   |        0  |       162   |      11 |
| TOML             |      1    |      10   |        7  |         1   |       2 |
| XSL              |      3    |    1116   |     1029  |        25   |      62 |
| XML              |   3464    |  180883   |   180134  |       452   |     297 |
| YAML             |     10    |    2151   |     1655  |       170   |     326 |
| Total            |   5413    | 1169015   |   904235  |     98287   |  166493 |

[^2]: Commits in the last 5 Years by gitstats
| Year | Commits (% of all) | Lines added | Lines removed |
| :--: | -----------------: | ----------: | ------------: |
| 2022 | 2891 (6.11%)       | 628760      | 755641        |
| 2021 | 4005 (8.46%)       | 1364990     | 1232564       |
| 2020 | 4357 (9.20%)       | 1914844     | 393243        |
| 2019 | 4299 (9.08%)       | 563890      | 273553        |
| 2018 | 3567 (7.54%)       | 2329146     | 5983747       |
