# Camlproto &nbsp; [![CI](https://github.com/Bannerets/camlproto/workflows/Build%20and%20test/badge.svg)](https://github.com/Bannerets/camlproto/actions?query=workflow%3A%22Build+and+test%22)

A portable and type-safe client implementation of Telegram's [MTProto][] protocol and [TL][] data serialization format.

[MTProto]: https://core.telegram.org/mtproto
[TL]: https://core.telegram.org/mtproto/TL

## Usage

Example to use the library with Telegram:

```ocaml
open Camlproto

module T = TLSchema.Telegram

module Client = Telegram.Client.Make(PlatformCaml)(TransportTcpFullCaml)

let prompt str = Lwt_io.(let%lwt () = write stdout str in read_line stdin)

let main () =
  (* api_id and api_hash can be obtained at https://my.telegram.org/ *)
  let%lwt phone_number = prompt "Enter your phone number: " in
  let%lwt api_id = prompt "Enter your api id: " in
  let api_id = int_of_string api_id in
  let%lwt api_hash = prompt "Enter your api hash: " in

  let%lwt t = Client.create () in

  let%lwt () = Client.init t (Telegram.Settings.create ~api_id ()) in
  let%lwt TL_auth_sentCode { phone_code_hash; _ } =
    Client.invoke t (module T.TL_auth_sendCode) {
      phone_number;
      api_id;
      api_hash;
      settings = TL_codeSettings {
        allow_flashcall = None;
        current_number = None;
        allow_app_hash = None;
        allow_missed_call = None;
        logout_tokens = None;
      }
    } in
  let%lwt phone_code = prompt "Enter the code: " in
  let%lwt [@warning "-8"] TL_auth_authorization { user; _ } =
    Client.invoke t (module T.TL_auth_signIn) {
      phone_number;
      phone_code_hash;
      phone_code;
    } in
  let TL_user { id; _ } | TL_userEmpty { id } = user in
  print_endline ("Signed as " ^ Int64.to_string id);
  Lwt.return_unit

let _ = Lwt_main.run (main ())
```

(see [examples/e02_telegram/](examples/e02_telegram/) and [examples/e01_mtproto/](examples/e01_mtproto/))

## TL <-> OCaml mapping

| TL                       | OCaml            |
|--------------------------|------------------|
| `int`                    | `int`            |
| `nat` (`#`)              | `int32`          |
| `long`                   | `int64`          |
| `string`                 | `string`         |
| `double`                 | `float`          |
| `int128`                 | `Cstruct.t`      |
| `int256`                 | `Cstruct.t`      |
| `bytes`                  | `Cstruct.t`      |
| `Bool`                   | `bool`           |
| `vector a`               | `'a list`        |
| Conditional definitions  | `'a option`      |

## "Transport components"

### Implemented

- tcp_full (ocaml, node.js)

- tcp_abridged (ocaml)

### In progress

- tcp_abridged (node.js)

### Not implemented

- websocket secure (browser)

- tcp_intermediate

- tcp_obfuscated2

- http

- https

- udp
