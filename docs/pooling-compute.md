# Pooling compute across your machines

Once there is more than one machine on a private network (a tailnet, a VPN, a home LAN), an agent can send work to whichever one has the capacity. A laptop can run a local model while a small server runs the scheduler. This page covers how to set that up without breaking access to the machines you already rely on. The examples use Tailscale terms; the ideas carry over to any mesh VPN with per-device identity.

## Two kinds of machine, two kinds of identity

Sort every machine into one of two kinds before you change anything:

1. **Servers**: always on, shared, nobody's personal device. Examples: an orchestrator VPS, a dedicated inference box, a hypervisor.
2. **Personal machines**: laptops and desktops that a person signs into, carries around, and uses as themselves.

Give servers a **device tag** (for example `tag:agents`). A tag makes the machine belong to the tag instead of to whoever enrolled it, so it doesn't break when that person leaves, and its keys don't expire on a person's schedule.

Never tag a personal machine. A tagged laptop stops counting as its owner, so every rule written for that person ("this user may SSH to X") silently stops matching when they connect from it. Removing the last tag is not an edit either: the machine has to be re-authenticated (`tailscale up --force-reauth`, repeating any non-default flags it asks for), and whoever signs in becomes its owner again.

## Reaching another person's server

A rule like "user A may SSH to user B's machines" is usually rejected. Tailscale only allows a *person* as the SSH destination when the source is that same person. The working pattern:

1. Define a tag and name its owners (the server's builder and the admin).
2. Apply the tag to the servers.
3. Write SSH rules with the tag as the destination: one rule for each person, listing only the Unix users they need.
4. Re-add the builder's own access. Their "my own devices" rule (`autogroup:self`) stops covering a machine once it's tagged.

Network reachability is a separate layer. A tailnet-wide allow rule lets every device reach every other on any port, whatever the SSH rules say.

## Two SSH servers, two rule sets

A mesh VPN's built-in SSH applies only on systems where that SSH server runs (typically Linux). A macOS machine running the stock menu-bar app answers SSH with the operating system's own server, which ignores the VPN's SSH rules and uses ordinary keys in `~/.ssh/authorized_keys`. When a connection fails, check which server answered:

- "tailnet policy does not permit you to SSH to this node": the VPN's SSH answered. Fix the policy, or the source machine's identity.
- "Permission denied (publickey)": the OS's SSH answered. Install the caller's public key.

## Using a personal machine's compute without tagging it

A laptop can still contribute: run the model server (Ollama, an MLX server, a GPU worker) on it, and have the scheduler call it at its tailnet name. Nothing about its identity changes. Three rules keep this sane:

1. The scheduler must treat a personal machine as **optional**. Laptops sleep and travel, so a job that needs one falls back to a server or retries later. It must never fail silently.
2. Bind the model server to the tailnet interface, not to all interfaces, so it isn't exposed on hotel Wi-Fi.
3. Record which machines offer which compute (model, memory, typical uptime) in the setup repo's machine map, next to the secrets map. Then the agent picks a target from a list instead of guessing.

## Putting credentials on a shared server

A shared server holds secrets for jobs that run without anyone signed in. Use a per-profile environment file with mode 600, add or replace one key at a time, and back up the file before each change. Move values point to point, from the password manager straight into the file (for example `op inject` piped over SSH), so a value never shows on a screen or in a transcript. An agent may be blocked from moving secrets into a shared store at all. When it is, the agent hands the human the exact command and the human runs it, which is the right split anyway.

Do per-person isolation on a shared server (separate Unix users or containers per profile) before personal credentials land there. Otherwise one profile's job can read another person's keys.
