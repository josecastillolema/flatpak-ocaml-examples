# How to Flatpak an OCaml application

Let's take a couple [OCaml](https://ocaml.org/) applications to illustrate [Flatpak](https://flatpak.org/) distributing and packaging leveraging the [Flatpak SDK Extension for OCaml](https://github.com/josecastillolema/org.freedesktop.Sdk.Extension.ocaml):
 - a hello world CLI application
 - a simple GTK program from the [Introduction to Gtk](https://v2.ocaml.org/learn/tutorials/introduction_to_gtk.html) OCaml tutorial

The SDK is only needed during the building phase.

**Table of contents**
- [How to Flatpak an OCaml application](#how-to-flatpak-an-ocaml-application)
  - [Creating a hello world CLI application](#creating-a-hello-world-cli-application)
    - [Flatpak manifest](#flatpak-manifest)
    - [Install and run the application](#install-and-run-the-application)
  - [Creating a simple GTK program](#creating-a-simple-gtk-program)
    - [Preparations](#preparations)
      - [Import GTK2 library](#import-gtk2-library)
      - [Wrapper script](#wrapper-script)
    - [Building from sources](#building-from-sources)
      - [Flatpak manifest](#flatpak-manifest-1)
      - [Install and run the application](#install-and-run-the-application-1)
    - [Building with opam](#building-with-opam)
      - [Source files](#source-files)
      - [Generate Flatpak manifest code](#generate-flatpak-manifest-code)
      - [Flatpak manifest](#flatpak-manifest-2)
      - [Install and run the application](#install-and-run-the-application-2)
  - [Publish the application](#publish-the-application)


## Creating a hello world CLI application

Create a hello world `example.ml`:
```ocaml
let () = print_endline "hello world"
```

You can compile your code with the OCaml compiler `ocamlopt` (no need to do so at this point):
```
$ ocamlopt -o example example.ml
$ ./example
hello world
```

### Flatpak manifest

Some things to note in the following manifest:
 - The SDK extension pointing to the [Flatpak SDK Extension for OCaml](https://github.com/josecastillolema/org.freedesktop.Sdk.Extension.ocaml) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.example.yaml#L9-L10)
 - The build options setting the `PATH` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.example.yaml#L13-L14)
 - Build some pre-requisistes using `dune` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.example.yaml#L17-L24)
 - Installing the application to `/app/bin` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.example.yaml#L33)

```yaml
app-id: flatpak.ocaml.example
runtime: org.freedesktop.Sdk
runtime-version: '22.08'
sdk: org.freedesktop.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.ocaml
command: example

build-options:
  append-path: /usr/lib/sdk/ocaml/bin

modules:
  - name: camlp-streams
    buildsystem: simple
    sources:
      - type: git
        url: https://github.com/ocaml/camlp-streams
        branch: trunk
    build-commands:
      - dune build

  - name: example
    buildsystem: simple
    sources:
      - type: file
        path: example.ml
    build-commands:
      - ocamlopt -o example example.ml
      - install -Dm755 example -t /app/bin/
```

### Install and run the application

Build and install the Flatpak package locally using `flatpak-builder`:
```
$ flatpak-builder --force-clean build-dir flatpak.ocaml.example.yaml
$ flatpak-builder --user --install --force-clean build-dir flatpak.ocaml.example.yaml
```

Run the application:
```
$ flatpak run flatpak.ocaml.example
hello world
```

The application has an installed size of 369.2 kB:
```
$ flatpak info flatpak.ocaml.example
          ID: flatpak.ocaml.example
         Ref: app/flatpak.ocaml.example/x86_64/master
        Arch: x86_64
      Branch: master
      Origin: example-origin
  Collection:
Installation: user
   Installed: 368.3 kB
     Runtime: org.freedesktop.Sdk/x86_64/22.08
         Sdk: org.freedesktop.Sdk/x86_64/22.08
      Commit: 02386f983e23b6d991651e86c1b1a55531fd4fa6504c730c938bc06033c19f40
     Subject: Export flatpak.ocaml.example
        Date: 2023-10-09 12:34:32 +0000
```

## Creating a simple GTK program

Let's use the simple `lablgtk` program from the [Introduction to Gtk](https://v2.ocaml.org/learn/tutorials/introduction_to_gtk.html) OCaml tutorial `simple.ml`:
```ocaml
open GMain
open GdkKeysyms

let locale = GtkMain.Main.init ()

let main () =
  let window = GWindow.window ~width:320 ~height:240
                              ~title:"Simple lablgtk program" () in
  let vbox = GPack.vbox ~packing:window#add () in
  window#connect#destroy ~callback:Main.quit;

  (* Menu bar *)
  let menubar = GMenu.menu_bar ~packing:vbox#pack () in
  let factory = new GMenu.factory menubar in
  let accel_group = factory#accel_group in
  let file_menu = factory#add_submenu "File" in

  (* File menu *)
  let factory = new GMenu.factory file_menu ~accel_group in
  factory#add_item "Quit" ~key:_Q ~callback: Main.quit;

  (* Button *)
  let button = GButton.button ~label:"Push me!"
                              ~packing:vbox#add () in
  button#connect#clicked ~callback: (fun () -> prerr_endline "Ouch!");

  (* Display the windows and enter Gtk+ main loop *)
  window#add_accel_group accel_group;
  window#show ();
  Main.main ()

let () = main ()
```

You can install the dependencies and compile your code (no need to do so at this point):
```
$ opam install lablgtk
$ ocamlfind ocamlopt -package lablgtk2 -linkpkg simple.ml -o simple
$ ./simple
Ouch!
```

This is what you should see when you run it:

![](img/simple.png)

### Preparations

#### Import GTK2 library

We need the GTK2 library, let's grab it from [Flathub shared-modules repo](https://github.com/flathub/shared-modules):
```
$ git submodule add https://github.com/flathub/shared-modules.git
```

#### Wrapper script

`LD_LIBRARY_PATH` is empty by default when running Flatpak applications ([flatpak/flatpak-builder#474](https://github.com/flatpak/flatpak-builder/issues/474)) so we will need a wrapper script `simple.sh` around the application:
```sh
export LD_LIBRARY_PATH=/app/lib
exec /app/bin/simple
```

### Building from sources

`lablgtk` can be built from its [sources](https://github.com/garrigue/lablgtk/archive/refs/tags/2.18.13.tar.gz).

#### Flatpak manifest

Some things to note in the following manifest:
 - The SDK extension pointing to the [Flatpak SDK Extension for OCaml](https://github.com/josecastillolema/org.freedesktop.Sdk.Extension.ocaml) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L9-L10)
 - Permissions for the Flatpak app to access X11 (`--share=ipc` and `--socket=fallback-x11`) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L12-L14)
 - The build options setting the `PATH` and some OCaml related environment variables [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L16-L19)
 - For some reason `debuginfo` was breaking `ocamlfind`, so it has been disabled [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L20)
 - We are importing GTK2 from the shared modules repository [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L23)
 - Installing some `lablgtk` pre-requisites: `ocamlfind` and `camlp-streams` (required only with OCaml>=5) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L25-L49)
 - `lablgtk` is build from its [sources](https://github.com/garrigue/lablgtk/archive/refs/tags/2.18.13.tar.gz) instead of using opam [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L51-L60)
 - Installing the application to `/app/bin` and `lablgtk` library to `/app/lib` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.build.yaml#L70-L71)


```yaml
app-id: flatpak.ocaml.lablgtk.build
runtime: org.freedesktop.Sdk
runtime-version: '22.08'
sdk: org.freedesktop.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.ocaml
command: simple.sh
finish-args:
  - --share=ipc
  - --socket=fallback-x11
build-options:
  append-path: /usr/lib/sdk/ocaml/bin:/app/share/runtime/ocaml
  env:
    CAML_LD_LIBRARY_PATH: /app/share/runtime/ocaml/lib:/usr/lib/sdk/ocaml/lib/ocaml:/usr/lib/sdk/ocaml/lib/ocaml/stublibs
    OCAMLFIND_CONF: /app/share/runtime/ocaml/findlib.conf
  no-debuginfo: true

modules:
  - shared-modules/gtk2/gtk2.json

  - name: ocamlfind
    buildsystem: simple
    sources:
      - type: git
        url: https://github.com/ocaml/ocamlfind
        branch: master
    build-commands:
      - mkdir -p ${FLATPAK_DEST}/share/runtime/ocaml
      - ./configure -bindir ${FLATPAK_DEST}/share/runtime/ocaml -config ${FLATPAK_DEST}/share/runtime/ocaml -no-topfind -sitelib ${FLATPAK_DEST}/share/runtime/ocaml/lib
      - make all
      - make install
    post-install:
      - ocamlfind printconf

  - name: camlp-streams
    buildsystem: simple
    sources:
      - type: git
        url: https://github.com/ocaml/camlp-streams
        branch: trunk
    build-commands:
      - dune build
      - dune install --prefix /app/share/runtime/ocaml
    post-install:
      - ocamlfind query camlp-streams

  - name: lablgtk
    buildsystem: simple
    sources:
      - type: archive
        url: https://github.com/garrigue/lablgtk/archive/refs/tags/2.18.13.tar.gz
        sha256: 7b9e680452458fd351cf8622230d62c3078db528446384268cd0dc37be82143c
    build-commands:
      - ./configure
      - make world
      - make old-install DESTDIR=/app/share/runtime/ocaml

  - name: simple
    buildsystem: simple
    sources:
      - type: file
        path: simple.ml
    build-commands:
      - mv /app/share/runtime/ocaml/usr/lib/sdk/ocaml/lib/ocaml/lablgtk2 /app/share/runtime/ocaml/lib
      - ocamlfind ocamlopt -package lablgtk2 -linkpkg simple.ml -o simple
      - install -Dm755 simple -t /app/bin/
      - mv /app/share/runtime/ocaml/usr/lib/sdk/ocaml/lib/ocaml/stublibs/dlllablgtk2.so /app/lib
    post-install:
      - rm -rf /app/share/runtime/ocaml

  - name: simple-wrapper
    buildsystem: simple
    sources:
      - type: file
        path: simple.sh
    build-commands:
      - cp simple.sh /app/bin
```

#### Install and run the application

Build and install the Flatpak package locally using `flatpak-builder`:
```
$ flatpak-builder --force-clean build-dir flatpak.ocaml.lablgtk.build.yaml
$ flatpak-builder --user --install --force-clean build-dir flatpak.ocaml.lablgtk.build.yaml
```

Run the application:
```
$ flatpak run flatpak.ocaml.lablgtk.build
Ouch!
```

This is what you should see when you run it:

![](img/simple.png)

The application has an installed size of 34.8 MB:
```
$ flatpak info flatpak.ocaml.lablgtk.build
          ID: flatpak.ocaml.lablgtk.build
         Ref: app/flatpak.ocaml.lablgtk.build/x86_64/master
        Arch: x86_64
      Branch: master
      Origin: build-origin
  Collection:
Installation: user
   Installed: 34.8 MB
     Runtime: org.freedesktop.Sdk/x86_64/22.08
         Sdk: org.freedesktop.Sdk/x86_64/22.08
      Commit: 6acda68f932e9728d942e976b6d6b94cab2929eb4c91b087f58792028f18d536
     Subject: Export flatpak.ocaml.lablgtk.build
        Date: 2023-12-19 09:29:13 +0000
```

### Building with opam

#### Source files

We can leverage the [OCaml Package Manager (opam)](https://opam.ocaml.org/) to install `lablgtk` into the Flatpak. For that, let's leverage [flatpak-opam-generator](https://github.com/josecastillolema/flatpak-opam-generator), a helper tool to automatically generate flatpak-builder source files from a `.json` generated by `opam tree`:
```
$ opam tree --json=lablgtk.json lablgtk
$ ./flatpak-opam-generator.py lablgtk.json > sources/lablgtk.json
```

The output sources file `sources/lablgtk.json` should look something like this:
```json
[
  {
    "type": "file",
    "url": "https://github.com/garrigue/lablgtk/archive/2.18.13.tar.gz",
    "name": "lablgtk.2.18.13",
    "md5": "d0a326b99475216cc22232e72c89415f",
    "dest": "cache/md5/d0",
    "dest-filename": "d0a326b99475216cc22232e72c89415f"
  },
  {
    "type": "file",
    "url": "https://github.com/ocaml/camlp-streams/archive/v5.0.1.tar.gz",
    "name": "camlp-streams.5.0.1",
    "md5": "afc874b25f7a1f13e8f5cfc1182b51a7",
    "dest": "cache/md5/af",
    "dest-filename": "afc874b25f7a1f13e8f5cfc1182b51a7"
  },
  {
    "type": "file",
    "url": "https://github.com/ocaml/dune/releases/download/3.10.0/dune-3.10.0.tbz",
    "name": "dune.3.10.0",
    "sha256": "9ff03384a98a8df79852cc674f0b4738ba8aec17029b6e2eeb514f895e710355",
    "dest": "cache/sha256/9f",
    "dest-filename": "9ff03384a98a8df79852cc674f0b4738ba8aec17029b6e2eeb514f895e710355"
  },
  {
    "type": "file",
    "url": "http://download.camlcity.org/download/findlib-1.9.6.tar.gz",
    "name": "ocamlfind.1.9.6",
    "md5": "96c6ee50a32cca9ca277321262dbec57",
    "dest": "cache/md5/96",
    "dest-filename": "96c6ee50a32cca9ca277321262dbec57"
  }
]
```

#### Generate Flatpak manifest code

Let's use the helper tool to generate the corresponding Flatpak manifest code:
```yaml
$ ./flatpak-opam-generator.py --generate lablgtk lablgtk.json
...
# Manifest code generated by flatpak-opam-generator
- name: lablgtk
  buildsystem: simple
  build-options:
    append-path: /usr/lib/sdk/ocaml/bin
    env:
      OPAMROOT: /usr/lib/sdk/ocaml
      OPAMSWITCH: /usr/lib/sdk/ocaml/switch
  sources:
    - sources/lablgtk.json
    - type: git
      branch: master
      url: https://github.com/ocaml/opam-repository
  build-commands:
    - ls -A --color=never | grep -Ev "cache|packages|repo" | xargs rm -rf
    - opam admin filter -y lablgtk.2.18.13 camlp-streams.5.0.1 dune.3.10.0 ocamlfind.1.9.6 
    - opam admin cache
    - opam repo add lablgtk .
    - opam install -y lablgtk.2.18.13 camlp-streams.5.0.1 dune.3.10.0 ocamlfind.1.9.6 
    - opam repo remove --all lablgtk
  post-install:
    - opam info --field name,all-installed-versions lablgtk
```

#### Flatpak manifest

Some things to note in the following manifest:
 - The SDK extension pointing to the [Flatpak SDK Extension for OCaml](https://github.com/josecastillolema/org.freedesktop.Sdk.Extension.ocaml) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L9-L10)
 - Permissions for the Flatpak app to access X11 (`--share=ipc` and `--socket=fallback-x11`) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L12-L14)
 - The build options setting the `PATH` and some OCaml related environment variables [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L15-L22)
 - We are importing GTK2 from the shared modules repository [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L25)
 - Creation of a new switch for installing `lablgtk` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L27-L44)
 - The manifest code generated by flatpak-opam-generator for `lablgtk` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L46-L62)
 - Installing the application to `/app/bin` and `lablgtk` library to `/app/lib` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L71-L72)
 - Removal of the ocaml switch at the post-install phase to limit the size of the resulting Flatpak application [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L73-L74)

```yaml
app-id: flatpak.ocaml.lablgtk.switch
runtime: org.freedesktop.Sdk
runtime-version: '22.08'
sdk: org.freedesktop.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.ocaml
command: simple.sh
finish-args:
  - --share=ipc
  - --socket=fallback-x11
build-options:
  append-path: /app/share/runtime/ocaml/bin:/app/share/runtime/ocaml/switch/_opam/bin:/usr/lib/sdk/ocaml/bin
  env:
    CAML_LD_LIBRARY_PATH: /app/share/runtime/ocaml/switch/_opam/lib/stublibs:/app/share/runtime/ocaml/switch/_opam/lib/ocaml/stublibs:/app/share/runtime/ocaml/switch/_opam/lib/ocaml
    OCAML_TOPLEVEL_PATH: /app/share/runtime/ocaml/switch/_opam/lib/toplevel
    OPAMROOT: /app/share/runtime/ocaml
    OPAMSWITCH: /app/share/runtime/ocaml/switch
    OPAM_SWITCH_PREFIX: /app/share/runtime/ocaml/switch/_opam

modules:
  - shared-modules/gtk2/gtk2.json

  - name: switch
    buildsystem: simple
    sources:
      - type: git
        branch: master
        url: https://github.com/ocaml/opam-repository
      - type: archive
        url: https://github.com/ocaml/ocaml/archive/refs/tags/5.1.0.tar.gz
        dest: switch
        sha256: 43a3ac7aab7f8880f2bb6221317be55319b356e517622fdc28359fe943e6a450
    build-commands:
      - mkdir -p ${FLATPAK_DEST}/share/runtime/ocaml
      - mv switch ${FLATPAK_DEST}/share/runtime/ocaml
      - opam init -y --bare --disable-sandboxing .
      - opam switch create -y --deps-only ${FLATPAK_DEST}/share/runtime/ocaml/switch
      - opam pin -y ${FLATPAK_DEST}/share/runtime/ocaml/switch
    post-install:
      - opam switch list

  # Manifest code generated by flatpak-opam-generator
  - name: lablgtk
    buildsystem: simple
    sources:
      - sources/lablgtk.json
      - type: git
        branch: master
        url: https://github.com/ocaml/opam-repository
    build-commands:
      - ls -A --color=never | grep -Ev "cache|packages|repo" | xargs rm -rf
      - opam admin filter -y lablgtk.2.18.13 camlp-streams.5.0.1 dune.3.10.0 ocamlfind.1.9.6
      - opam admin cache
      - opam repo add lablgtk .
      - opam install -y lablgtk.2.18.13 camlp-streams.5.0.1 dune.3.10.0 ocamlfind.1.9.6
      - opam repo remove --all lablgtk
    post-install:
      - opam info --field name,all-installed-versions lablgtk

  - name: simple
    buildsystem: simple
    sources:
      - type: file
        path: simple.ml
    build-commands:
      - ocamlfind ocamlopt -package lablgtk2 -linkpkg simple.ml -o simple
      - install -Dm755 simple -t /app/bin/
      - cp /app/share/runtime/ocaml/switch/_opam/lib/stublibs/dlllablgtk2.so /app/lib
    post-install:
      - rm -rf /app/share/runtime/ocaml

  - name: simple-wrapper
    buildsystem: simple
    sources:
      - type: file
        path: simple.sh
    build-commands:
      - cp simple.sh /app/bin
```

#### Install and run the application

Build and install the Flatpak package locally using `flatpak-builder`:
```
$ flatpak-builder --force-clean build-dir flatpak.ocaml.lablgtk.switch.yaml
$ flatpak-builder --user --install --force-clean build-dir flatpak.ocaml.lablgtk.switch.yaml
```

Run the application:
```
$ flatpak run flatpak.ocaml.lablgtk.switch
Ouch!
```

This is what you should see when you run it:

![](img/simple.png)

The application has an installed size of 11.3 MB:
```
$ flatpak info flatpak.ocaml.lablgtk.switch
          ID: flatpak.ocaml.lablgtk.switch
         Ref: app/flatpak.ocaml.lablgtk.switch/x86_64/master
        Arch: x86_64
      Branch: master
      Origin: lablgtk-origin
  Collection:
Installation: user
   Installed: 11.3 MB
     Runtime: org.freedesktop.Sdk/x86_64/22.08
         Sdk: org.freedesktop.Sdk/x86_64/22.08
      Commit: b3159cf05f2d89d7caaae3bc7bc6fd6a2ab70d5195374b4222a3cbedf8f635b0
      Parent: 499a0892394a54c93976d422aa533563b6a1b1212fd9036fbc4fcbdd023f87dd
     Subject: Export flatpak.ocaml.lablgtk.switch
        Date: 2023-10-16 12:01:01 +0000
```


## Publish the application

Info on how to later publish the application on [Flathub](https://flathub.org/) can be found [here](https://docs.flathub.org/docs/for-app-authors/submission/).
