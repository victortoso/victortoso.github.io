---
title: Generating Cgo for libvirt
date: 2023-01-01
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
which raises the need of `libvirt-go-module` to be patched constantly with Cgo to wrap around
libvirt's new additions plus the actual Go code that will expose it to Go applications.

Reducing this work as much as possible would help with the maintenance of `libvirt-go-module` while
making it easier to keep go bindings up to date with libvirt.

Luckily, libvirt's API is [thoroughly documented][] in a format that we can take advantage of in
order to generate almost all of the Cgo code, leaving developers able to focus on how to expose
libvirt's new features.

[libvirt-go-module]: https://gitlab.com/libvirt/libvirt-go-module/
[Cgo code]: https://go.dev/blog/cgo
[virConnectOpen()]: https://libvirt.org/html/libvirt-libvirt-host.html#virConnectOpen
[Libvirt]: https://libvirt.org/
[thoroughly documented]: https://libvirt.org/html/index.html

# The input

Libvirt's `scripts/apibuild.py` parses the source code to gather all the metadata it can to build
its API documentation which produces four XMLs, one per module. The `<api>` section has two
subsections, `<files>` and `<symbols>`. The symbols part is what we are mostly interested in order
to generate those Cgo wrappers.

Taking the `virDomainSetLaunchSecurityState()` function as example. We have know when it was
introduced, what are the return and arguments type and the correct argument order.

{{< highlight xml >}}
    <function name='virDomainSetLaunchSecurityState'
              file='libvirt-domain'
              module='libvirt-domain'
              version='8.0.0'>
      <info><![CDATA[
Set a launch security secret in the guest's memory. The guest must be
in a paused state, e.g. in state VIR_DOMIAN_PAUSED as reported by
virDomainGetState. On success, the guest can be transitioned to a
running state. On failure, the guest should be destroyed.]]></info>
      <return type='int'
              info='-1 in case of failure, 0 in case of success.'/>
      <arg name='domain'
           type='virDomainPtr'
           info='a domain object'/>
      <arg name='params'
           type='virTypedParameterPtr'
           info='pointer to launch security parameter objects'/>
      <arg name='nparams'
           type='int'
           info='number of launch security parameters'/>
      <arg name='flags'
           type='unsigned int'
           info='currently used, set to 0.'/>
    </function>
{{< /highlight >}}

# Adding version metadata

As one of `libvirt-go-module`'s goals is to be able to compile and run in platforms supported by
libvirt itself. This means we need to be aware of libvirt's version at compile and running time. At
compile time, we might need libvirt symbols that are not included in older versions of its headers;
At run time, to avoid calling functions that are not yet implement, like `SetIdentity` below:

{{< highlight go "linenos=table, linenostart=582" >}}
// See also https://libvirt.org/html/libvirt-libvirt-host.html#virConnectSetIdentity
func (c *Connect) SetIdentity(ident *ConnectIdentity, flags uint32) error {
    if C.LIBVIR_VERSION_NUMBER < 5008000 {
        return makeNotImplementedError("virConnectSetIdentity")
    }
    /* implements the function  */
{{< /highlight >}}

Starting in libvirt's v8.3.0, the version information was included in all the symbols exported in
XML. This involved patching `scripts/apibuild.py` to parse a `Since:` tag from that symbol's
comments plus adding the `Since: $version`  to all exported symbols. [This last bit alone][] made me
learn a few new `git` and `regular expressions` tricks.

[This last bit alone]: https://listman.redhat.com/archives/libvir-list/2022-April/230357.html

# The code generator

The logic of code generator is relatively simple:

1. Locate the path of XMLs with `pkg-config --variable`
2. Loading the XMLs into Go Structus
3. Preparing the loaded data (e.g: sorting, do some calculations)
4. Opening output files
5. Generate code by running the templates

The `encoding/xml` Go module makes it very easy to [Unmarshal][] XML into structs and I decided to
use `text/template` to write Cgo templates for each symbol. The base `API struct` below is used to
load the XML and as argument for each of the templates.

{{< highlight go "linenos=table, linenostart=43" >}}
type API struct {
	XMLName   xml.Name      `xml:"api"`
	Name      string        `xml:"name,attr"`
	Files     []APIFile     `xml:"files>file"`
	Macros    []APIMacro    `xml:"symbols>macro"`
	Typedefs  []APITypedef  `xml:"symbols>typedef"`
	Enums     []APIEnum     `xml:"symbols>enum"`
	Structs   []APIStruct   `xml:"symbols>struct"`
	Functions []APIFunction `xml:"symbols>function"`
	Functypes []APIFunctype `xml:"symbols>functype"`
	Variables []APIVariable `xml:"symbols>variable"`
}
{{< /highlight >}}

Just to give an example, the [template for enums][] has less than 20 lines of code to generate more
than 3400 lines of [C header][] with proper version checking.

[template for enums]: https://gitlab.com/libvirt/libvirt-go-module/-/blob/master/gen/api_generated_enums.h.tmpl
[C header]: https://gitlab.com/libvirt/libvirt-go-module/-/blob/master/libvirt_generated_enums.h

{{< highlight go-text-template "linenos=table, linenostart=31" >}}
#pragma once

{{- with .Enums }}
    {{- $lastTypeDefined := "" }}
    {{- range . }}
        {{- if ne $lastTypeDefined .Type }}
            {{- $lastTypeDefined = .Type }}

/* enum {{ .Type }} */
        {{- end }}
#  if !LIBVIR_CHECK_VERSION({{ getVersionMajor .Version }}, {{ getVersionMinor .Version }}, {{ getVersionMicro .Version }})
#    define {{ .Name }} {{ getEnumString . }}
#  endif
    {{- end }}
{{ end }}

{{< /highlight >}}

Enums are generated in blocks like:

{{< highlight bash "linenos=table, linenostart=68" >}}
/* enum virConnectBaselineCPUFlags */
#  if !LIBVIR_CHECK_VERSION(1, 1, 2)
#    define VIR_CONNECT_BASELINE_CPU_EXPAND_FEATURES (1 << 0)
#  endif
#  if !LIBVIR_CHECK_VERSION(1, 2, 14)
#    define VIR_CONNECT_BASELINE_CPU_MIGRATABLE (1 << 1)
#  endif
{{< /highlight >}}

[Unmarshal]: https://pkg.go.dev/encoding/xml#Unmarshal

# Compiling without libvirt headers

The title of the [actual merge request][] was `Generate Wrapper code that uses dlopen/dlsym` but
after a few exchanges, we decided to take the feature of loading libvirt at runtime as a compile
time option and disabled by default.

There are two major benefit of using [dlopen/dlsym][].
1. Build the application consuming libvirt-go-module without worrying about libvirt as build-time
   dependency. For example, this can benefit [KubeVirt][] to reduce the amount of rpm's they need to
   track for their Bazel builds.
2. Run the application consuming libvirt-go-module on any system and only install and load libvirt
   if needed. This can benefit applications where the binary offers libvirt as an option between
   other technologies.

To use the `dlopen/dlsym` feature, one needs to build `libvirt-go-module` with `libvirt_dlopen` tag
(e.g: `go build -tags libvirt_dlopen`).

Let's take a look at the generated functions of `virDomainSetLaunchSecurityState()` that we put the
XML in `The input` section.

First, the [static version][]:

{{< highlight c "linenos=table, hl_lines=12, linenostart=3378" >}}
int
virDomainSetLaunchSecurityStateWrapper(virDomainPtr domain,
                                       virTypedParameterPtr params,
                                       int nparams,
                                       unsigned int flags,
                                       virErrorPtr err)
{
    int ret = -1;
#if !LIBVIR_CHECK_VERSION(8, 0, 0)
    setVirError(err, "Function virDomainSetLaunchSecurityState not available prior to libvirt version 8.0.0");
#else
    ret = virDomainSetLaunchSecurityState(domain,
                                          params,
                                          nparams,
                                          flags);
    if (ret < 0) {
        virCopyLastError(err);
    }
#endif
    return ret;
}
{{< /highlight >}}

At build time we check if libvirt's version is new enough to contain that function. At runtime we
either call it or set the error appropriately. This is basically how it was working before but now
it is generated.

The [dlopen version][]:

{{< highlight c "linenos=table, hl_lines=26, linenostart=5375" >}}
typedef int
(*virDomainSetLaunchSecurityStateType)(virDomainPtr domain,
                                       virTypedParameterPtr params,
                                       int nparams,
                                       unsigned int flags);

int
virDomainSetLaunchSecurityStateWrapper(virDomainPtr domain,
                                       virTypedParameterPtr params,
                                       int nparams,
                                       unsigned int flags,
                                       virErrorPtr err)
{
    int ret = -1;
    static virDomainSetLaunchSecurityStateType virDomainSetLaunchSecurityStateSymbol;
    static bool once;
    static bool success;

    if (!libvirtSymbol("virDomainSetLaunchSecurityState",
                       (void**)&virDomainSetLaunchSecurityStateSymbol,
                       &once,
                       &success,
                       err)) {
        return ret;
    }
    ret = virDomainSetLaunchSecurityStateSymbol(domain,
                                                params,
                                                nparams,
                                                flags);
    if (ret < 0) {
        virCopyLastErrorWrapper(err);
    }
    return ret;
}
{{< /highlight >}}

Here we have to actually define the function type and then load the function into
`virDomainSetLaunchSecurityStateSymbol` variable, before we can call it.

The `libvirtSymbol()` function is the helper to `dlopen()` the library once and `dlsym()` its
symbol. Let's take a peek at it.

{{< highlight c "linenos=table, hl_lines=27, linenostart=75" >}}
bool
libvirtSymbol(const char *name,
              void **symbol,
              bool *once,
              bool *success,
              virErrorPtr err)
{
    char *errMsg;

    if (!libvirtLoad(err)) {
        return *success;
    }

    if (*once) {
        if (!*success) {
            // Set error for successive calls
            char msg[100];
            snprintf(msg, 100, "Failed to load %s", name);
            setVirError(err, msg);
        }
        return *success;
    }

    // Documentation of dlsym says we should use dlerror() to check for failure
    // in dlsym() as a NULL might be the right address for a given symbol.
    // This is also the reason to have the @success argument.
    *symbol = dlsym(handle, name);
    if ((errMsg = dlerror()) != NULL) {
        setVirError(err, errMsg);
        *once = true;
        return *success;
    }
    *once = true;
    *success = true;
    return *success;
}
{{< /highlight >}}

It is more error handling than anything else :)

This process is similar for `lxc`, `qemu` and `admin` drivers with all this code being generated.

[actual merge request]: https://gitlab.com/libvirt/libvirt-go-module/-/merge_requests/7
[dlopen/dlsym]: https://man7.org/linux/man-pages/man3/dlopen.3.html
[KubeVirt]: http://kubevirt.io/
[static version]: https://gitlab.com/libvirt/libvirt-go-module/-/blob/master/libvirt_generated_functions_static_domain.go#L3378
[dlopen version]: https://gitlab.com/libvirt/libvirt-go-module/-/blob/master/libvirt_generated_functions_dlopen_domain.go#L5375


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
