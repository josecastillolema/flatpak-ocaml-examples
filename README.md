# How to Flatpak an Ocaml application

Let's take a hello world [Ocaml](https://ocaml.org/) application to illustrate [Flatpak](https://flatpak.org/) distributing and packaging leveraging the [Flatpak SDK Extension for OCaml](https://github.com/josecastillolema/org.freedesktop.Sdk.Extension.ocaml).

## Creating a hello world app

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

## Flatpak manifest
```yaml
app-id: flatpak.ocaml.example
runtime: org.freedesktop.Sdk
runtime-version: '22.08'
sdk: org.freedesktop.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.ocaml
command: example

build-options:
  append-path: /usr/lib/sdk/ocaml/bin:/usr/lib/sdk/ocaml/5.0.0/_opam/bin
  env:
    CAML_LD_LIBRARY_PATH: /usr/lib/sdk/ocaml/5.0.0/_opam/lib/stublibs:/usr/lib/sdk/ocaml/5.0.0/_opam/lib/ocaml/stublibs:/usr/lib/sdk/ocaml/5.0.0/_opam/lib/ocaml
    OCAML_TOPLEVEL_PATH: /usr/lib/sdk/ocaml/5.0.0/_opam/lib/toplevel
    OPAMROOT: /usr/lib/sdk/ocaml
    OPAMSWITCH: /usr/lib/sdk/ocaml/5.0.0
    OPAM_SWITCH_PREFIX: /usr/lib/sdk/ocaml/5.0.0/_opam

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

## Install and run the application

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

Info on how to later publish the application on [Flathub](https://flathub.org/) can be found [here](https://docs.flathub.org/docs/for-app-authors/submission/).
