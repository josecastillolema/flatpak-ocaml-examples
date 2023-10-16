# How to Flatpak an OCaml application

Let's take a couple [OCaml](https://ocaml.org/) applications to illustrate [Flatpak](https://flatpak.org/) distributing and packaging leveraging the [Flatpak SDK Extension for OCaml](https://github.com/josecastillolema/org.freedesktop.Sdk.Extension.ocaml):
 - a hello world CLI application
 - a simple GTK program from the [Introduction to Gtk](https://v2.ocaml.org/learn/tutorials/introduction_to_gtk.html) OCaml tutorial

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
 - The build options setting the `PATH` and some OCaml related environment variables [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.example.yaml#L13-L20)
 - Installing the application to `/app/bin` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.example.yaml#L30)

```yaml
app-id: flatpak.ocaml.example
runtime: org.freedesktop.Sdk
runtime-version: '22.08'
sdk: org.freedesktop.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.ocaml
command: example

build-options:
  append-path: /usr/lib/sdk/ocaml/bin:/usr/lib/sdk/ocaml/switch/_opam/bin
  env:
    CAML_LD_LIBRARY_PATH: /usr/lib/sdk/ocaml/switch/_opam/lib/stublibs:/usr/lib/sdk/ocaml/switch/_opam/lib/ocaml/stublibs:/usr/lib/sdk/ocaml/switch/_opam/lib/ocaml
    OCAML_TOPLEVEL_PATH: /usr/lib/sdk/ocaml/switch/_opam/lib/toplevel
    OPAMROOT: /usr/lib/sdk/ocaml
    OPAMSWITCH: /usr/lib/sdk/ocaml/switch
    OPAM_SWITCH_PREFIX: /usr/lib/sdk/ocaml/switch/_opam

modules:
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
   Installed: 369.2 kB
     Runtime: org.freedesktop.Sdk/x86_64/22.08
         Sdk: org.freedesktop.Sdk/x86_64/22.08
      Commit: 02386f983e23b6d991651e86c1b1a55531fd4fa6504c730c938bc06033c19f40
     Subject: Export flatpak.ocaml.example
        Date: 2023-10-09 12:34:32 +0000
```

Info on how to later publish the application on [Flathub](https://flathub.org/) can be found [here](https://docs.flathub.org/docs/for-app-authors/submission/).


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
$ ocamlfind ocamlc -g -package lablgtk2 -linkpkg simple.ml -o simple
$ ./simple
Ouch!
```

This is what you should see when you run it:

![](img/simple.png)

### Preparations

#### Source files

We need to use the [OCaml Package Manager (opam)](https://opam.ocaml.org/) to install `lablgtk` into the Flatpak. For that, let's leverage [flatpak-opam-generator](https://github.com/josecastillolema/flatpak-opam-generator), a helper tool to automatically generate flatpak-builder source files from a `.json` generated by `opam tree`:
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

### Flatpak manifest

Some things to note in the following manifest:
 - The SDK extension pointing to the [Flatpak SDK Extension for OCaml](https://github.com/josecastillolema/org.freedesktop.Sdk.Extension.ocaml) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L9-L10)
 - The `writable-sdk` option uses a writable copy of the SDK for `/usr` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L11). The less intrusive `ensure-writable` option does not seem to work in this scenario [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L31-L32).

   > It is possible to avoid this by creating a new switch for installing `lablgtk`, see [flatpak.ocaml.lablgtk.switch.yaml](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.switch.yaml#L27-L41) example. In this case, the resulting installed app size is 11.2 MB.

 - Permissions for the Flatpak app to access X11 (`--share=ipc` and `--socket=fallback-x11`) [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L13-L15)
 - The build options setting the `PATH` and some OCaml related environment variables [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L16-L24)
 - We are importing GTK2 from the shared modules repository [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L27)
 - The manifest code generated by flatpak-opam-generator for `lablgtk` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L29-L47)
 - Installing the application to `/app/bin` and `lablgtk` library to `/app/lib` [&#8629;](https://github.com/josecastillolema/flatpak-ocaml-examples/blob/main/flatpak.ocaml.lablgtk.yaml#L56-L57)


```yaml
app-id: flatpak.ocaml.lablgtk
runtime: org.freedesktop.Sdk
runtime-version: '22.08'
sdk: org.freedesktop.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.ocaml
writable-sdk: true
command: simple.sh
finish-args:
  - --share=ipc
  - --socket=fallback-x11
build-options:
  append-path: /usr/lib/sdk/ocaml/bin:/usr/lib/sdk/ocaml/switch/_opam/bin
  env:
    CAML_LD_LIBRARY_PATH: /usr/lib/sdk/ocaml/switch/_opam/lib/stublibs:/usr/lib/sdk/ocaml/switch/_opam/lib/ocaml/stublibs:/usr/lib/sdk/ocaml/switch/_opam/lib/ocaml
    OCAML_TOPLEVEL_PATH: /usr/lib/sdk/ocaml/switch/_opam/lib/toplevel
    OPAMROOT: /usr/lib/sdk/ocaml
    OPAMSWITCH: /usr/lib/sdk/ocaml/switch
    OPAM_SWITCH_PREFIX: /usr/lib/sdk/ocaml/switch/_opam
    OPAMLOGS: /tmp

modules:
  - shared-modules/gtk2/gtk2.json

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
      - ocamlfind ocamlc -g -package lablgtk2 -linkpkg simple.ml -o simple
      - install -Dm755 simple -t /app/bin/
      - cp /usr/lib/sdk/ocaml/switch/_opam/lib/stublibs/dlllablgtk2.so /app/lib

  - name: simple-wrapper
    buildsystem: simple
    sources:
      - type: file
        path: simple.sh
    build-commands:
      - cp simple.sh /app/bin
```

### Install and run the application

Build and install the Flatpak package locally using `flatpak-builder`:
```
$ flatpak-builder --force-clean build-dir flatpak.ocaml.lablgtk.yaml
$ flatpak-builder --user --install --force-clean build-dir flatpak.ocaml.lablgtk.yaml
```

Run the application:
```
$ flatpak run flatpak.ocaml.lablgtk
Ouch!
```

This is what you should see when you run it:

![](img/simple.png)

The application has an installed size of 10.4 MB:
```
$ flatpak info flatpak.ocaml.lablgtk
          ID: flatpak.ocaml.lablgtk
         Ref: app/flatpak.ocaml.lablgtk/x86_64/master
        Arch: x86_64
      Branch: master
      Origin: lablgtk-origin
  Collection: 
Installation: user
   Installed: 11.2 MB
     Runtime: org.freedesktop.Sdk/x86_64/22.08
         Sdk: org.freedesktop.Sdk/x86_64/22.08
      Commit: b3159cf05f2d89d7caaae3bc7bc6fd6a2ab70d5195374b4222a3cbedf8f635b0
      Parent: 499a0892394a54c93976d422aa533563b6a1b1212fd9036fbc4fcbdd023f87dd
     Subject: Export flatpak.ocaml.lablgtk
        Date: 2023-10-16 12:01:01 +0000
```

Info on how to later publish the application on [Flathub](https://flathub.org/) can be found [here](https://docs.flathub.org/docs/for-app-authors/submission/).
