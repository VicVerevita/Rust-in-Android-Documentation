#	***Android Services in Rust - pocket dict*** 



###	Interacting with services on the device 

Android comes with a few commands to allow interacting with services on the device. Try:

```shell
$ adb shell dumpsys --help # listing and dumping services
$ adb shell service --help # sending commands to services for testing

```



# AIDL Interfaces

Here is an example AIDL interface:

```java
interface ITeleport {
        void teleport(Location baz, float speed);
        String getName();
    }
    
```



### Importing Types

In Rust Backend:

```rust
use my_package::aidl::my::package::IFoo;

```



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



### Debugging API for services:

In Rust Backend:

```rust
fn dump(&self, mut file: &File, args: &[&CStr]) -> binder::Result<()>
    
```



## Thread Management

In Rust Backend:

```rust
binder::ProcessState::start_thread_pool();
binder::add_service(“myservice”, my_service_binder).expect(“Failed to register service?”);
binder::ProcessState::join_thread_pool();

```



## 	Reserved Names

**For Rust, the field or type is renamed using the "raw identifier" syntax, accessible using the ```r#``` prefix.**




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



#### `nullable`

`nullable` declares that the value of the annotated entity may not be provided.
This annotation can only be attached to method return types, method parameters, and parcelable fields.

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



#### `backing`

The `Backing` annotation specifies the storage type of an AIDL enum type.

```java
@Backing(type="int")
enum Color { RED, BLUE, }
```



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

  

#### `hide` in comments

The AIDL compiler recognizes `@hide` in comments and passes it through to Java output for metalava to pickup. This comment ensures that the Android build system knows that AIDL APIs are not SDK APIs.



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





# Getting Started - Download ~2-4 hours :/



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



### Launching emulator

Note: I use the server's noVNC, because the `ssh -X` does not support screen mirror for the emulator.

Now the steps are simple:

```bash
$ cd ~/WORKING_DIRECTORY

$ source build/envsetup.sh

// Mentioned above why you use `sdk` and not `aosp`
$ lunch sdk_car_x86_64-userdebug

$ make	
// or `m` in subfolders
```

To run the emulator:

```bash
$ emulator
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



### For Android.bp - soong build

https://ci.android.com/builds/submitted/8206919/linux/latest/view/soong_build.html





Now we finally move to...

# Rust in Android

