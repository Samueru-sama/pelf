# PELF
<!-- [![Go Report Card](https://goreportcard.com/badge/github.com/xplshn/pelf)](https://goreportcard.com/report/github.com/xplshn/pelf) -->
![License](https://img.shields.io/github/license/xplshn/pelf)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/xplshn/pelf)

PELF (Pack an ELF) is a toolset designed to simplify the process of turning your binaries into single-file executables, similar to AppImages. The format used by PELF is called `.AppBundle` or `.blob`. The PELF files are portable across systems of the same architecture and ABI. Architecture and LIBC-independent bundles can be achieved using Wrappers.

If you only intend on using .AppBundles, not necesarily work with them, you don't need any of this. You can get started by simply executing the bundle. The helper daemon is optional.
![SpectrWM window manager AppBundle/.blob that contains all of my X utils including Wezterm](https://github.com/xplshn/pelf/assets/114888778/b3b99c24-825d-4be0-a1c8-ea9433776692)
As you can see, I have my window manager and all of my X utilities, including my terminal (Wezterm) as a single file, named SpectrWM.AppBundle. You can also see the concept of overlays in action, the `ani-cli` binary inside of the mpv.AppBundle, will have access to the ROFI binary packed in my SpectrWM.AppBundle, because it will be running as a child of that process. There is PATH and LD_LIBRARY_PATH inheritance.

## Tools Included
### `pelf` ![pin](assets/pin.svg)
`pelf` is the main tool used to create an `.AppBundle` from your binaries. It takes your executable files and packages them into a single file for easy distribution and execution.

**Usage:**
```sh
./pelf [ELF_SRC_PATH] [DST_PATH.blob] <--add-library [LIB_PATH]> <--add-binary [BIN_PATH]> <--add-metadata [icon128x128.xpm|icon128x128.png|icon.svg|app.desktop]> <--add-arbitrary [DIR|FILE]>
```

### `pelf_linker`
`pelf_linker` is a utility that facilitates access to binaries inside an `.AppBundle` for other programs. It ensures that binaries and dependencies within the (open, overlayed) bundles are accesible to external programs. This relies on the concept of Bundle Overlays.

**Usage:**
```sh
pelf_linker [--export] [binary]
```

### `pelf_extract`
`pelf_extract` allows you to extract the contents of a PELF bundle to a specified folder. This can be useful for inspecting the contents of a bundle or for modifying its contents.

**Usage:**
```sh
./pelf_extract [input_file] [output_directory]
```

### `pelfd` ![pin](assets/pin.svg)
`pelfd` is a daemon written in Go that automates the "installation" of `.AppBundle` files. It automatically puts the metadata of your .AppBundles in the appropriate directories, such as `.local/share/applications` and `.local/share/icons`. So that the .AppBundles you put in ~/Programs (for example) will appear in your menus.

**Usage:**
```sh
pelfd &
```

## Overlaying Bundles ![pin](assets/pin.svg)
One of the key features of PELF is its ability to overlay bundles on top of each other. This means that programs inside one bundle can access binaries and libraries from other bundles. For example, if you bundle `wezterm` as a single file and add `wezterm-mux-server` to the same bundle using `--add-binary`, programs run by `wezterm` will be able to see all of the binaries and libraries inside the `wezterm` bundle.

This feature is particularly powerful because you can stack an infinite number of PELF bundles. For instance:

- `spectrwm.blob` contains `dmenu`, `xclock`, and various X utilities like `scrot` and `rofi`.
- `wezterm.blob` contains some Lua programs and utilities.
- `mpv.blob` contains `ani-cli`, `ani-skip`, `yt-dlp`, `fzf`, and `curl`.

Using the `pelf_linker`, the `mpv.blob` can access binaries inside `spectrwm.blob` as well as its own binaries. By doing `mpv.blob --pbundle_link ani-cli`, you can launch an instance of the `ani-cli` included in the bundle, as well as ensure that it can access other utilities in the linked/stacked bundles.

## Installation ![pin](assets/pin.svg)
To install the PELF toolkit, follow these steps:

1. Clone the repository:
    ```sh
    git clone https://github.com/xplshn/pelf.git ~/Programs
    cd ~/Programs
    rm LICENSE README.md
    cd cmd && go build && mv ./pelfd && cd - # Build the helper daemon
    rm -rf ./cmd ./examples
    ```
2. Add ~/Programs to your $PATH in your .profile or .shrc (.kshrc|.ashrc|.bashrc)

## Usage Examples ![pin](assets/pin.svg)
### Creating an `.AppBundle`
To create an `.AppBundle` from your binaries, use the `pelf` tool:
```sh
./pelf /usr/bin/wezterm wezterm.AppBundle --add-binary /usr/bin/wezterm-mux-server --add-metadata /usr/share/applications/wezterm.desktop --add-metadata ./wezterm128x128.png --add-metadata ./wezterm128x128.svg --add-metadata ./wezterm128x128.xpm
```

### Linking an `.AppBundle`
To make the binaries inside of your (open & overlayed) PELFs visible and usable to other programs, you can use the `pelf_linker` tool:
```sh
pelf_linker ytfzf
```

### Extracting an `.AppBundle`
To extract the contents of an `.AppBundle` to a folder, use the `pelf_extract` tool:
```sh
./pelf_extract openArena.AppBundle ./openArena_bundleDir
```

### Running the `pelfd` Daemon
To start the `pelfd` daemon and have it automatically manage your `.AppBundle` files:
```sh
pelfd &
```
On the first-run, it will create a config file which you can modify:
**~/.config/pelfd.json**, this is how your config would look after the first run:
```json
{
  "options": {
    "directories_to_walk": [
      "~/Programs"
    ],
    "probe_interval": 90,
    "icon_dir": "/home/anto/.local/share/icons",
    "app_dir": "/home/anto/.local/share/applications",
    "probe_extensions": [
      ".AppBundle",
      ".blob"
    ]
  },
  "tracker": {}
}
```
- `"directories_to_walk"`: This is an array of directories that the `pelfd` daemon will monitor for `.AppBundle` or `.blob` files. By default, it is set to `["~/Programs"]`, meaning the daemon will only check for AppBundles in the `~/Programs` directory. You can add more directories to this array if you want the daemon to watch multiple locations.
- `"probe_interval"`: This specifies the interval in seconds at which the `pelfd` daemon will check the specified directories for new or modified AppBundles. By default, it is set to `90` seconds.
- `"icon_dir"`: This is the directory where icons extracted from .AppBundles will be copied to `pelfd`. By default, it is set to `"~/.local/share/icons"`, which is a common location for application icons on modern Unix clones.
- `"app_dir"`: This is the directory where the desktop files extracted from .AppBundles will be copied to by `pelfd`. By default, it is set to `"~/.local/share/applications"`. `.desktop` files provide information about the application, such as its name, icon, and command to execute. By default, it is set to `"~/.local/share/applications"`, which is the standard location for application shortcuts on modern Unix clones.
- `"probe_extensions"`: This is an array of file extensions that `pelfd` will look for when probing the specified directories. By default, it is set to `[".AppBundle", ".blob"]`, meaning the daemon will only consider files with these extensions as AppBundles. (CASE-SENSITIVE)
- The "tracker" object in the config file is used to store information about the tracked AppBundles, this way, when an AppBundle is removed, its files can be safely "uninstalled", etc.

## Contributing
Contributions to PELF are welcome! If you find a bug or have a feature request, please open an Issue. For direct contributions, fork the repository and submit a pull request with your changes.

## License
PELF is licensed under the 3BSD License. See the [LICENSE](LICENSE) file for more details.

### Special thanks to:
- [Chris DeBoy](https://codeberg.org/chris_deboy)
- [Jade Mandelbrot+ for writting the README.md](https://hf.co/chat/assistant/667646ec7dd4a03abaf916f4)
