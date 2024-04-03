# PowerShell Bridge for Bash Completions

Bridge to enable bash completions to be run from within PowerShell.

Commands like `kubectl` allow you to export command completion logic for use in bash. As these same commands get ported to Windows and/or used within PowerShell, porting the dynamic completion logic becomes challenging for the project maintainers. This project is an attempt to make a bridge so things "just work."

## Installation

### Prerequisites for Windows

**Make sure you have `bash.exe` in your path or that you have Git for Windows installed.** If `bash.exe` isn't in the path, the version shipping with Git for Windows will be used.

## Prerequisites for MacOS

**You need a newer bash than the one shipped with MacOS.** If you haven't ever upgraded bash, likely you've still got 3.2 from 2007. Use Homebrew to get a newer version. `brew install bash`

### From PowerShell Gallery

`Install-Module -Name "PSBashCompletions"`

### Manual Install

1. Put the `PSBashCompletions` folder in your PowerShell module folder (e.g., `C:\Users\username\Documents\WindowsPowerShell\Modules` or `/Users/username/.local/share/powershell/Modules`.
2. `Import-Module PSBashCompletions`

## Usage

1. Locate the completion script for the bash command. You may have to export this like:

   ```powershell
   ((kubectl completion bash) -join "`n") | Set-Content -Encoding ASCII -NoNewline -Path kubectl_completions.sh
   ```

   **Make sure the completion file is ASCII.** Some exports (like `kubectl`) come out as UTF-16 with CRLF line endings. Windows `bash` may see this as a binary file that can't be interpreted which results in no completions happening. Using `join` and `Set-Content` ensures the completions script will load in bash.

2. Run the `Register-BashArgumentCompleter` cmdlet to register the command you're expanding and the location of the completions.

   Example:

   ```powershell
   Register-BashArgumentCompleter "kubectl" C:\completions\kubectl_completions.sh
   ```

3. If you use PowerShell aliases, register the completer for your aliases as well.

   Example:

   ```powershell
   Set-Alias kc kubectl
   Register-BashArgumentCompleter "kc" C:\completions\kubectl_completions.sh
   ```

## How It Works

The idea is to register a PowerShell argument completer that will manually invoke the bash completion mechanism and return the output that bash would have provided. Basically, that means:

- A bridge script (`bash_completion_bridge.sh`) is used to load up the exported bash completions and manually invoke the completion functions.
- From PowerShell, locate bash, locate the bridge script, and register a completer that ties those together.
- When PowerShell invokes the completer, the completer arguments are taken and passed to the bash bridge.
- The bash bridge executes the actual completion and returns the results, which are passed back through to PowerShell.

It won't be quite as fast as if it was all running native but it means you can use provided bash completions instead of having to re-implement in PowerShell.

## Demo Expansions

In the `Demo` folder there are some expansions to try:

- `Register-KubectlCompleter.ps1` - Expansions for `kubectl`
- `Register-GitCompleter.ps1` - Expansions for `git`
- `Register-EchoTestCompleter.ps1` - Simple echo script that shows completion parameters in bash; use `echotest` as the command to complete and whatever else after it to simulate command lines.

## Troubleshooting

There are lots of moving pieces, so if things aren't working there isn't "one answer" as to why.

**First, run `Register-BashArgumentCompleter` with the `-Verbose` flag.** When you do that, you'll get a verbose line that shows the actual `bash.exe` command that will be used to generate completions. You can copy that and run it yourself to see what happens.

The command will look something like this in Windows:

`&"C:\Windows\system32\bash.exe" "/c/Users/username/Documents/WindowsPowerShell/Modules/PSBashCompletions/1.2.2/bash_completion_bridge.sh" "/c/completions/kubectl_completions.sh" "<url-encoded-command-line>"`

On MacOS, it'll look like this:

`&"/usr/local/bin/bash" "/Users/username/.local/share/powershell/Modules/PSBashCompletions/1.2.2/bash_completion_bridge.sh" "/Users/username/.config/powershell/bash-completion/kubectl_completions.sh" "<url-encoded-command-line>"`

The last parameter is what you can play with - it's a URL-encoded version of the whole command line being completed (with `%20` as space, not `+`).

Say you're testing `kubectl` completions and want to see what would happen if you hit TAB after `kubectl c`. You'd run:

`&"C:\Windows\system32\bash.exe" "/c/Users/username/Documents/WindowsPowerShell/Modules/PSBashCompletions/1.0.0/bash_completion_bridge.sh" "/c/completions/kubectl_completions.sh" "kubectl%20c"`

If it works, you'd see a list like:

```text
certificate
cluster-info
completion
config
convert
cordon
cp
create
```

**If it generates an error, that's the error to troubleshoot.**

Common things that can go wrong:

- Bash isn't found or the path to bash is wrong.
- You have an old bash version and need to upgrade.
- Your completion script isn't found or the path is wrong.
- Your completion script isn't ASCII with `\n` line endings. UTF-16 doesn't get read as text by Windows `bash`; UTF-8 with a byte-order mark also sometimes causes problems. Even Windows `\r\n` line endings may cause trouble with parsing sometimes. Keeping it ASCII and `\n` is the safest way to ensure compatibility.
- You have something in your bash profile that's interfering with the completions.
- You're trying to use a completion that isn't compatible with the command on your OS. This happens with `git` completions - for example, on Windows you need to use the completion script that comes with Git for Windows, not the Linux version.
- The completions rely on other commands or functions that aren't available/loaded. If the completion script isn't self-contained, things won't work. For example, the `kubectl` completions actually call `kubectl` to get resource names in some completions. If bash can't find `kubectl`, the completion won't work.
- The default Windows Subsystem for Linux (WSL) distro is non-standard. See the "Known Issues" section below for details on this.

## Add to Your Profile

After installing the module you can set the completions to be part of your profile. One way to do that:

- Create a folder for completions in your PowerShell profile folder, like: `C:\Users\username\OneDrive\Documents\WindowsPowerShell\bash-completion`
- Save all of your completions in there (e.g., `kubectl_completions.sh`).
- Add a script block to your `Microsoft.PowerShell_profile.ps1` that looks for `bash` (or Git for Windows) and conditionally registers completions based on that. (This will avoid errors if you sync your PowerShell profile to machines that might not have bash.)

```powershell
$enableBashCompletions = ($Null -ne (Get-Command bash -ErrorAction Ignore)) -or ($Null -ne (Get-Command git -ErrorAction Ignore))

if ($enableBashCompletions) {
  Import-Module PSBashCompletions
  $completionPath = [System.IO.Path]::Combine([System.IO.Path]::GetDirectoryName($profile), "bash-completion")
  Register-BashArgumentCompleter kubectl "$completionPath/kubectl_completions.sh"
  Register-BashArgumentCompleter git "$completionPath/git_completions.sh"
}
```

## Known Issues

### Flags Don't Autocomplete

PowerShell doesn't appear to pass _flags_ to custom argument completers. So, say you tried to do this:

`kubectl apply -<TAB>`

Note the `TAB` after the dash `-`. In bash you'd get completions for the flag like `-f` or `--filename`. PowerShell doesn't seem to call a custom argument completer for flags. (If you know how to make that work [let me know!](https://github.com/tillig/ps-bash-completions/issues))

### Recursive Command Calls Don't Work

Some completions (like `kubectl`) actually call themselves to generate completions. For example, if you do:

`kubectl get pod -n <TAB>`

...then `kubectl` will actually be called to go get the list of your namespaces to try to autocomplete the remote values for you. In cases like this, you may see that PowerShell will instead do something weird like duplicate the flag...

`kubectl get pod -n -n`

...or it may do nothing at all. If you run the troubleshooting command line, you may see an error message, possibly something cryptic like `compopt: not currently executing completion function`.

This happens because we're not _actually in bash doing the completion_, we're manually invoking the completion and the fake-out isn't deep enough. I don't know how to fix that, since `kubectl` or whatever might not actually be installed in bash for you - it may be a Windows/PowerShell thing. Even if it was, getting all the levels working is beyond my ken. (If you know how to make that work [let me know!](https://github.com/tillig/ps-bash-completions/issues))

### Non-Standard WSL Distro

On running `Register-BashArgumentCompleter` in Windows you may see messages that look like this:

> `An error occurred mounting one of your file systems. Please run 'dmesg' for more details.`

or like this:

```text
<3>WSL (11) ERROR: CreateProcessEntryCommon:370: getpwuid(0) failed 2
<3>WSL (11) ERROR: CreateProcessEntryCommon:374: getpwuid(0) failed 2
<3>WSL (11) ERROR: CreateProcessEntryCommon:577: execvpe /bin/sh failed 2
<3>WSL (11) ERROR: CreateProcessEntryCommon:586: Create process not expected to return
```

This is typically because the default distro is somewhat non-standard, likely the Docker distro. You can check this by running `wsl -l` to list the distros you have installed.

Most likely you'll see something like this:

```text
PS> wsl -l
Windows Subsystem for Linux Distributions:
docker-desktop-data (Default)
Ubuntu
docker-desktop
```

You need to change the default distro to be something standard like `Ubuntu` so that the `bash` command works: `wsl -s Ubuntu`

[For more information, check out this issue thread.](https://github.com/microsoft/WSL/issues/5923)
