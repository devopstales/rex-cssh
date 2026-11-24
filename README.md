# rex-cssh

Cluster SSH for Rex, a terminal multiplexer made by [Superlogical](https://www.superlogical.com/),
in the spirit of [tmux-cssh](https://github.com/zinic/tmux-cssh). It opens one Rex pane per
host, each running `ssh`, and replicates what you type in **any** pane to all
the others.

## Requirements

- Rex with the `rex` CLI on your `PATH`
- Python 3.9 or later (standard library only)
- `ssh`

## Install

```sh
chmod +x rex-cssh
ln -s "$PWD/rex-cssh" /usr/local/bin/rex-cssh   # optional
```

## Usage

```sh
rex-cssh [options] [user@]host[:port] ...
rex-cssh -c CLUSTER [options]
```

Examples:

```sh
rex-cssh web1 web2 admin@db1:2222          # three panes
rex-cssh -u deploy -i ~/.ssh/id_ed25519 web1 web2 web3
rex-cssh -c web                            # hosts from a cluster
rex-cssh -c web,db -l prod                 # several clusters, custom label
rex-cssh -x 'uptime' -t web1 web2          # run a command on each host
rex-cssh -d web1 web2 web3                 # dry run: print the ssh commands
```

If you run it from inside a Rex session, it opens a new tab in that session.
Outside Rex, or with `-n`, it creates a new session instead. It then tries to
switch your Rex app to that session. This needs Remote Control turned on in the
app's Settings under Rex Server. If it is off, select the session yourself.

The panes are laid out in an even grid, for example 5 hosts become two rows of
3 and 2.

### Options

| Option | Description |
| --- | --- |
| `-c, --cluster NAME[,NAME]` | Add hosts from the cluster file |
| `-u, --user USER` | Default ssh user (a `user@` on a host wins) |
| `-p, --port PORT` | Default ssh port (a `:port` on a host wins) |
| `-i, --identity FILE` | ssh identity file |
| `-o, --option OPT` | ssh `-o` option, repeatable |
| `-x, --remote-cmd CMD` | Command to run on each host |
| `-t, --tty` | Force a tty (`ssh -t`) when used with `-x` |
| `-l, --label LABEL` | Tab or session label (default `cssh-<cluster or first host>`) |
| `-n, --new-session` | Open a new session (named by `-l`) instead of a new tab |
| `--ssh CMD` | ssh command to use (default `ssh`) |
| `-d, --dry-run` | Print what would run, then exit |

IPv6 hosts use brackets: `user@[::1]:22`.

## Keys

Type these in any pane. The prefix is `Ctrl-]`.

| Keys | Action |
| --- | --- |
| `Ctrl-]` `t` | Toggle broadcast for all panes. Labels show `[solo]` while it is off, so you can type in one pane. |
| `Ctrl-]` `q` | Close all the host panes |
| `Ctrl-]` `]` | Send a literal `Ctrl-]` to ssh |

When a host's `ssh` exits cleanly, its pane closes. If it fails, the pane stays
open until you press Enter so you can read the error.

## Cluster file

Clusters live in `~/.rex-cssh.conf`. Override the path with
`$REX_CSSH_CONF`.

```
# name: host host user@host:port ...
web: web1 web2 web3
db:  admin@db1 admin@db2:2222
```

## How it works

Rex has no built-in "synchronize panes", and it can't report what you type in a
pane. So every pane runs `ssh` through a small wrapper (`rex-cssh --pane`, an
internal mode) that:

1. runs `ssh` on a pty,
2. passes your keystrokes to it, and
3. forwards them to the other wrappers over unix datagram sockets in a
   temporary directory under `/tmp`.

Received keystrokes are written only to the local `ssh`, so nothing loops.
The directory is removed when the last pane exits; if the session is killed
from outside, a leftover directory is swept by the next run (after one hour).

## Limitations

- Keystrokes typed before every pane has started may not reach all panes.
- The grid layout is fixed, because Rex has no "tiled" command.
- Broadcast is all or nothing. There is no per-pane exclusion yet beyond
  toggling it off and typing in a single pane.
