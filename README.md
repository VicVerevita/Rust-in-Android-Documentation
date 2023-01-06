#	***Android Services in Rust - pocket dict*** 

&nbsp;

###	Interacting with services on the device 

Android comes with a few commands to allow interacting with services on the device. Try:

```shell
$ adb shell dumpsys --help # listing and dumping services
$ adb shell service --help # sending commands to services for testing

```

&nbsp;

# AIDL Interfaces

Here is an example AIDL interface:

```java
interface ITeleport {
        void teleport(Location baz, float speed);
        String getName();
    }
    
```

&nbsp;

### Importing Types

In Rust Backend:

```rust
use my_package::aidl::my::package::IFoo;

```

&nbsp;

### Implementing Types

In Rust Backend:

```rust
    use aidl_interface_name::aidl::my::package::IFoo::{BnFoo, IFoo};
    use binder;

    /// This struct is defined to implement IRemoteService AIDL interface.
    pub struct MyFoo;

    impl Interface for MyFoo {}

    impl IFoo for MyFoo {
        fn doFoo(&self) -> binder::Result<()> {
           ...
           Ok(())
        }
    }
    
```

&nbsp;

###	Registering and getting services

In Rust Backend:

```rust
use myfoo::MyFoo;
use binder;
use aidl_interface_name::aidl::my::package::IFoo::BnFoo;

fn main() {
    binder::ProcessState::start_thread_pool();
    // [...]
    let my_service = MyFoo;
    let my_service_binder = BnFoo::new_binder(
        my_service,
        BinderFeatures::default(),
    );
    binder::add_service("myservice", my_service_binder).expect("Failed to register service?");
    // Does not return - spawn or perform any work you mean to do before this call.
    binder::ProcessState::join_thread_pool()
}

```

&nbsp;

### Debugging API for services:

In Rust Backend:

```rust
fn dump(&self, mut file: &File, args: &[&CStr]) -> binder::Result<()>
    
```

&nbsp;

## Thread Management

In Rust Backend:

```rust
binder::ProcessState::start_thread_pool();
binder::add_service(“myservice”, my_service_binder).expect(“Failed to register service?”);
binder::ProcessState::join_thread_pool();

```

&nbsp;

## 	Reserved Names

**For Rust, the field or type is renamed using the "raw identifier" syntax, accessible using the ```r#``` prefix.**

&nbsp;


##	Registering a service

Each service is created and registered with ```servicemanager```. Registration often occurs in a file named main.cpp, but the implementation can vary. The registration usually looks something like this:

```cpp
using android::defaultServiceManager;

defaultServiceManager()->addService(serviceName, service);

```

Registration is sometimes abstracted by ```BinderService::publish``` or ```BinderService::instantiate```, which call the above code.

To register a service as dynamic, replace its registration code with the following:

```cpp
#include <binder/LazyServiceRegistrar.h>

using android::binder::LazyServiceRegistrar;

auto lazyRegistrar = LazyServiceRegistrar::getInstance();
lazyRegistrar.registerService(service, serviceName);

```

```servicemanager``` communicates with ```LazyServiceRegistrar``` to shut down services based on their reference counts.

&nbsp;

## Dynamic AIDL

#### Configuring a service’s init .rc file

To run a service dynamically, add the following options to the service’s init `.rc` file after the leading `service <name> <cmd>` line.

```
interface aidl serviceName
disabled
oneshot	
```

These options do the following:

- `interface aidl serviceName`: Allows `servicemanager` to find the service. If the service uses multiple interfaces, declare each interface on its own line. These names must be exactly what `servicemanager` expects and may differ from the process name.
- `disabled`: Prevents the service from automatically starting at boot.
- `oneshot`: Prevents the service from automatically restarting each time it is stopped.

&nbsp;

## Annotations

Similar to Java:

```java
@AnnotationName(argument1=value, argument2=value) AidlEntity

@AnnotationName AidlEntity
```

**These annotations are not the same as the Java annotations, although they look very similar. Users can't define custom AIDL annotations; the annotations are all pre-defined. **
Below is the list of pre-defined AIDL annotations:

| Annotations                | Added in Android version |
| :------------------------- | :----------------------- |
| `nullable`                 | 7                        |
| `utf8InCpp`                | 7                        |
| `VintfStability`           | 11                       |
| `UnsupportedAppUsage`      | 10                       |
| `Hide`                     | 11                       |
| `Backing`                  | 11                       |
| `JavaOnlyStableParcelable` | 11                       |
| `JavaDerive`               | 12                       |
| `JavaPassthrough`          | 12                       |
| `FixedSize`                | 12                       |
| `Descriptor`               | 12                       |

&nbsp;

#### `nullable`

`nullable` declares that the value of the annotated entity may not be provided.
This annotation can only be attached to method return types, method parameters, and parcelable fields.

&nbsp;

#### `VintfStability`

`VintfStability` declares that a user-defined type (interface, parcelable, and enum) can be used across the system and vendor domains. See [AIDL for HALs](https://source.android.com/docs/core/architecture/aidl/aidl-hals) for more about system-vendor interoperability.

```java
@VintfStability
interface IFoo {
    ....
}

@VintfStability
parcelable Data {
    ....
}

@VintfStability
enum Type {
    ....
}
```

When a type is annotated with `VintfStability`, any other type that is referenced in the type should also be annotated as such. In the example below, `Data` and `IBar` should both be annotated with `VintfStability`.

```java
@VintfStability
interface IFoo {
    void doSomething(in IBar b); // references IBar
    void doAnother(in Data d); // references Data
}

@VintfStability // required
interface IBar {...}

@VintfStability // required
parcelable Data {...}
```

In addition, the AIDL files defining types annotated with `VintfStability` can only be built using the `aidl_interface` Soong module type, with the `stability` property set to `"vintf"`.

&nbsp;

#### `backing`

The `Backing` annotation specifies the storage type of an AIDL enum type.

```java
@Backing(type="int")
enum Color { RED, BLUE, }
```

&nbsp;

#### `JavaOnlyStableParcelable`

`JavaOnlyStableParcelable` marks a parcelable declaration (not definition) as stable so that it can be referenced from other stable AIDL types.

```java
parcelable Data { // Data is a structured parcelable.
    int x;
    int y;
}

parcelable AnotherData { // AnotherData is also a structured parcelable
    Data d; // OK, because Data is a structured parcelable
}
```

If the parcelable was unstructured (or just declared), then it can't be referenced.

```java
parcelable Data; // Data is NOT a structured parcelable

parcelable AnotherData {
    Data d; // Error
}
```

`JavaOnlyStableParcelable` lets you to override the check when the parcelable you are referencing is already safely available as part of the Android SDK.

```java
@JavaOnlyStableParcelable
parcelable Data;

parcelable AnotherData {
    Data d; // OK
}
```

&nbsp;

#### `JavaDerive`

`JavaDerive` automatically generates methods for parcelable types in the Java backend.

```java
@JavaDerive(equals = true, toString = true)
parcelable Data {
  int number;
  String str;
}
```

The annotation requires additional parameters to control what to generate. The supported parameters are:

- `equals=true` generates `equals` and `hashCode` methods.

- `toString=true` generates `toString` method that prints the name of the type and fields. For example: `Data{number: 42, str: foo}`

  &nbsp;

#### `hide` in comments

The AIDL compiler recognizes `@hide` in comments and passes it through to Java output for metalava to pickup. This comment ensures that the Android build system knows that AIDL APIs are not SDK APIs.

&nbsp;

## AIDL Fuzzing

The fuzzer behaves as a client for the remote service by importing/invoking it through the generated stub:

Using Rust API:

```rust
#![allow(missing_docs)]
#![no_main]
#[macro_use]
extern crate libfuzzer_sys;

use binder::{self, BinderFeatures, Interface};
use binder_random_parcel_rs::fuzz_service;

fuzz_target!(|data: &[u8]| {
    let service = BnTestService::new_binder(MyService, BinderFeatures::default());
    fuzz_service(&mut service.as_binder(), data);
});
```

&nbsp;

&nbsp;

# Getting Started - Download ~2-4 hours, if not more :/

&nbsp;

### Initializing Repo

1. Create an empty directory to hold your working files. Give it any name you like:

   ```bash
   mkdir WORKING_DIRECTORY
   cd WORKING_DIRECTORY
   ```

   Note: you name the folder yourself...

   

2. Configure Git with your real name and email address. To use the Gerrit code-review tool, you need an email address that's connected with a [registered Google account](https://www.google.com/accounts). Ensure that this is a live address where you can receive messages. The name that you provide here shows up in attributions for your code submissions.

   ```bash
   git config --global user.name Your Name
   git config --global user.email you@example.com
   ```

3. Run `repo init` to get the latest version of Repo with its most recent bug fixes. You must specify a URL for the manifest, which specifies where the various repositories included in the Android source are placed within your working directory.

   ```bash
   repo init -u https://android.googlesource.com/platform/manifest -b android-12.0.0_r2
   ```

   **You need to use `sdk_car_x86_64-userdebug` instead of `aosp_car_x86_64-userdebug`. The first one will generate all you need for the AVD. The second one just creates a pure GSI.**

   

4. To download the Android source tree to your working directory from the repositories as specified in the default manifest, run:

   ```bash
   repo sync
   ```

   To speed syncs, pass the `-c` (current branch) and `-jthreadcount` flags:

   ```bash
   repo sync -c -j8
   ```

&nbsp;

### Launching emulator

Note: I use the server's noVNC, because the `ssh -X` does not support screen mirror for the emulator.

Now the steps are simple:

```bash
cd ~/WORKING_DIRECTORY

source build/envsetup.sh

// Mentioned above why you use `sdk` and not `aosp`
lunch sdk_car_x86_64-userdebug

make	
// or `m` in subfolders
```

To run the emulator:

```bash
emulator
	-wipe-data
	-selinux permissive
	-sysdir $ANDROID_PRODUCT_OUT
	-datadir $ANDROID_PRODUCT_OUT
	-kernel $ANDROID_PRODUCT_OUT/kernel-ranchu
	-ramdisk $ANDROID_PRODUCT_OUT/ramdisk-qemu.img
	-system $ANDROID_PRODUCT_OUT/system-qemu.img
	-vendor $ANDROID_PRODUCT_OUT/vendor-qemu.img
	-encryption-key $ANDROID_PRODUCT_OUT/encryptionkey.img
```

&nbsp;

### For Android.bp - soong build

https://ci.android.com/builds/submitted/8206919/linux/latest/view/soong_build.html

&nbsp;

&nbsp;

&nbsp;

Now we finally move to...

# Rust in Android

&nbsp;

### **Android Rust Modules**

As a general principle, `rust_*` module definitions adhere closely to `cc_*` usage and expectations. The following is an example of a module definition for a Rust binary:

```go
rust_binary {
    name: "hello_rust",
    crate_name: "hello_rust",
    srcs: ["src/hello_rust.rs"],
    host_supported: true,
}
```

&nbsp;

#### Important common properties

* `name`

  > The name of your module. 
  > Must be unique across most `Android.bp` module type.
  > It is used by default as the output name.

&nbsp;

* `stem`

  > **Optional**
  >
  > Provides control over the output name (mentioned above, default as `name`).
  >
  > Use the `stem` function when you can't set the module name to the desired output filename. For example, the `rust_library` for the `log` crate is named [`liblog_rust`](https://android.googlesource.com/platform/external/rust/crates/log/+/6ff3cc55db6651188710a55831548c3e7ff796ad/Android.bp), because a [`liblog cc_library`](https://android.googlesource.com/platform/system/logging/+/b2cb35cc4d5da1840c8db4c133e49a551c5c694f/liblog/Android.bp) already exists. Using the `stem` property in this case ensures that the output file is named `liblog.*` instead of `liblog_rust.*`.

&nbsp;

* `srcs`

  > Contains the source file that is the entry point to your module (usually `main.rs` or `lib.rs`).
  >
  > When possible, avoid this usage for platform code; see [Source Generators](https://source.android.com/docs/setup/build/rust/building-rust-modules/source-code-generators/source-code-gen-intro) for more information.

  &nbsp;

* `crate_name`

  > `crate_name` sets the crate name metadata through the `rustc` `--crate_name` flag. For modules which produce libraries, this **must** match the expected crate name used in source. For example, if the module `libfoo_bar` is referenced in source as `extern crate foo_bar`, then this **must** be crate_name: "foo_bar".
  >
  > It's **required** for modules which produce Rust libraries (such as `rust_library` `rust_ffi`, `rust_bindgen`, `rust_protobuf`, and `rust_proc_macro`). 
  >
  > They enforce `rustc` requirements on the relationship between `crate_name` and the output (mentioned at `name` and `stem`).
  >
  > For more information, see the [Library Modules](https://source.android.com/docs/setup/build/rust/building-rust-modules/library-modules#notable-rust-library-properties) section.

  &nbsp;

* `lints`

  > The [rustc linter](https://doc.rust-lang.org/rustc/lints/levels.html) is run by default for all module types except source generators.
  >
  > Possible values for such lint sets are as follows:
  >
  > - `default` the default set of lints, depending on the module's location
  > - `android` the strictest lint set that applies to all Android platform code
  > - `vendor` a relaxed set of lints applied to vendor code
  > - `none` to ignore all lint warnings and errors

  &nbsp;

* `clippy_lints`

  > [clippy linter](https://github.com/rust-lang/rust-clippy)
  >
  > Same as `lints`, but only a few sets of lints are defined which are used to validate module source. 

  ##### ***Tip:*** You can set linters to `none` while developing to avoid linter errors. However, during review you might need to justify having committed modules with less-strict lints.

  &nbsp;

* `edition`

  > Straightforward. Rust edition - `2015` or `2018`.

  &nbsp;

* `flags`

  >`flags` contains a string list of flags to pass to `rustc` during compilation.

  &nbsp;

* `id_flags`

  > `ld-flags` contains a string list of flags to pass to the linker when compiling source. These are passed by the `-C linker-args` rustc flag. `clang` is used as the linker front-end, invoking `lld` for the actual linking.
  >
  > &nbsp;
  >
  > **Note:** Because these are passed to Clang, treat them as **Clang** flags. Prefix flags to pass to `lld` with the usual `-Wl` Clang prefix.

  &nbsp;

* `features`

  >`features` is a string list of features that must be enabled during compilation. This is passed to rustc by `--cfg 'feature="foo"'`. 
  >
  >Most features are additive, so in many cases this consists of the full features set required by all dependent modules. 
  >
  >However, in cases where features are exclusive of one another, define additional modules in any build files that provide conflicting features.
  >
  >&nbsp;
  >
  >**Note:** Unlike in a Cargo build, there's no “default” set of features which are implicitly enabled. The "default" feature is a Cargo-special keyword; unless the code explicitly handles a feature called "default", *setting "default" has no impact*.

  &nbsp;

* `cfgs`

  > This contains a list of strings of `cfg` flags to be enabled during compilation.
  >
  > This is passed to `rustc` by `--cfg foo` and `--cfg "fizz=buzz"`.
  >
  > The build system automatically sets certain `cfg` flags in particular situations, listed below:
  >
  > - Modules built as a dylib will have the `android_dylib` cfg set.
  > - Modules which would use the VNDK will have the `android_vndk` cfg set. This is similar to the [`__ANDROID_VNDK__`](https://source.android.com/docs/core/architecture/vndk/build-system#conditional-cflags) definition for C++.

  &nbsp;

* `strip`

  > This controls whether and how the output file is stripped.
  >
  > Valid values include `none` to disable stripping, and `all` to strip everything, including the mini debuginfo.
  >
  > Additional values can be found in the [Soong Modules Reference](https://ci.android.com/builds/latest/branches/aosp-build-tools/targets/linux/view/soong_build.html).

  &nbsp;

* `host_supported`

  > For device modules, the `host_supported` parameter indicates whether the module should also provide a host variant.

&nbsp;

#### Defining library dependencies

| Property Name       | Description                                                  |
| :------------------ | :----------------------------------------------------------- |
| `rustlibs`          | List of `rust_library` modules that are also dependencies. Use this as your preferred method of declaring dependencies, as it allows the build system to select the preferred linkage. (See [When linking against Rust libraries](https://source.android.com/docs/setup/build/rust/building-rust-modules/android-rust-modules#when-linking-against), below) |
| `rlibs`             | List of `rust_library` modules that must be statically linked as `rlibs`. (Use with caution; see [When linking against Rust libraries](https://source.android.com/docs/setup/build/rust/building-rust-modules/android-rust-modules#when-linking-against), below.) |
| `dylibs`            | List of `rust_library` modules to be dynamically linked as `dylibs`. (Use with caution; see [When linking against Rust libraries](https://source.android.com/docs/setup/build/rust/building-rust-modules/android-rust-modules#when-linking-against), below.) |
| `shared_libs`       | List of `cc_library` modules which must be dynamically linked as shared libraries. |
| `static_libs`       | List of `cc_library` modules which must be statically linked as static libraries. |
| `whole_static_libs` | List of `cc_library` modules which should be statically linked as static libraries and included whole in the resulting library. For `rust_ffi_static` variants, `whole_static_libraries` will be included in the resulting static library archive. For `rust_library_rlib` variants, `whole_static_libraries` libraries will be bundled into the resulting `rlib` library.<br /><br />**Tip:** This prevents you from redeclaring dependents' dependencies for this library. However, in some cases this can cause bloat if the static library is duplicated within a dependency tree. Wholly included static libraries in `rlibs` will be wholly included in dependent `dylibs`, which provides another source of potential bloat. |

*When linking against Rust libraries*, as a best practice, do so using the `rustlibs` property rather than `rlibs` or `dylibs` unless you have a specific reason to do so. This allows the build system to select the correct linkage based on what the root module requires, and it reduces the chance that a dependency tree contains both the `rlib` and `dylib` versions of a library (which will cause compilation to fail).

&nbsp;

**Unsupported and limited support build features**

Soong's Rust offers limited support for `vendor` and `vendor_ramdisk` images and snapshots. However, `staticlibs`, `cdylibs`, `rlibs`, and `binaries` are supported. For vendor image build targets, the `android_vndk` `cfg` property is set. You can use this in code if there are differences between the system and vendor targets. `rust_proc_macros` aren't captured as part of vendor snapshots; if these are depended on, ensure you appropriately version-control them.

Product, VNDK, and Recovery images aren't supported.

&nbsp;

