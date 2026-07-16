VEHIR INSTALL
=============

Read this entire guide before doing anything. Every step assumes you
know what comes after it. If you skip ahead and something breaks,
start over from the top.

Before starting, check for files or directories that would conflict with
the paths used below. Do not overwrite or merge existing content without
the user's approval.

The release archive is not a binary-only distribution. It contains the
complete IPM source tree together with the native artifacts needed to
bootstrap that exact generation. The source is available under
`generations/N/sources/` before any Vehir component is started.

1. DOWNLOAD AND EXTRACT

   Choose a published generation from the releases page. Throughout this
   guide, replace `N` with that generation number.

   Download N.tar.gz from the releases page:
   https://github.com/grigoriitropin/vehir-platform/releases

   Then extract:

   tar -xzf ~/N.tar.gz -C ~


2. READ THE GUIDE

   You MUST read ~/generations/N/etc/DEV-GUIDE.md before proceeding. This is
   not optional. If you skip this and something breaks, this is the first file
   you will need.


3. CREATE THE VEHIR STORE ROOT

   Ask the user where the Vehir store should be installed. Recommend the
   default:

       ~/.address-objects-using-content-hashes

   If they choose a different directory, use that path throughout the
   following steps.

   Always convert the chosen path to an absolute form before using it in
   any configuration file. If the user gives a path with ~ (tilde),
   resolve it to the actual home directory path first.

   The chosen location is the Vehir store root. After this step its
   relevant layout is:

       <INSTALL_ROOT>/
       ├── current -> generations/N
       └── generations/
           └── N/

   `current` must be inside `<INSTALL_ROOT>`. It is the stable entry point
   to the active generation: the service, MCP configuration, and persistent
   agent instruction all refer to `<INSTALL_ROOT>/current/...`.
   Generations are immutable and reconciliation may promote a later one.
   `current` is the atomic selector for the active known-good generation.
   Do not point services directly at `generations/N` and do not place
   `current` outside the store root.

   Move only the extracted `generations/` directory into the store root.
   Other store directories are created automatically when reconciliation
   needs them.

   Then:

   mkdir -p <chosen-location>
   mv ~/generations <chosen-location>/


4. CREATE current SYMLINK

   ln -sf generations/N <chosen-location>/current


5. FIX HARDCODED PATHS

   The generation was built on a machine where the home directory was
   /home/vehir. Config files under <chosen-location>/current/etc/ may
   contain this path and will not work on your machine.

   Search for the hardcoded path:

   grep -r '/home/vehir' <chosen-location>/current/etc/

   Replace every occurrence with your actual home directory. The exact
   command depends on your username — verify before running.


6. SYSTEMD SERVICE

   Check whether `~/.config/systemd/user/` exists. If it does not, create it.
   Do not modify any existing files in that directory.

   Place at ~/.config/systemd/user/vehir-kernel.service :

   [Unit]
   Description=Vehir kernel host-boot seed (firmware launcher; auto-restart on crash)
   After=network.target

   [Service]
   Type=simple
   ExecStart=<chosen-location>/current/libexec/loader '{"launch-kernel-objects-following-bootconfig":{"config":"<chosen-location>/current/etc/boot-launch.json"}}'
   Restart=always
   RestartSec=2

   [Install]
   WantedBy=default.target

   Then:

   systemctl --user daemon-reload
   systemctl --user enable --now vehir-kernel
   systemctl --user status vehir-kernel


7. MCP INTEGRATION

   The MCP server listens on 127.0.0.1, port configured in
   <chosen-location>/current/etc/mcp/listen.json
   (default: 8765 = edge_port_hi*256 + edge_port_lo = 34*256 + 61).

   All Vehir capabilities are exposed as a single MCP tool (vehir-shell).

   -- Identify your agent --

   If you know which agent you are running in, read the corresponding file
   in <chosen-location>/current/etc/integrations/mcp/ completely before
   changing any configuration. If you do not know, ask the user which agent
   they are using, then read the corresponding file.

   Integration-guide configurations are examples of valid Vehir settings,
   not files to overwrite verbatim. Preserve unrelated MCP servers,
   permissions, settings, and project-specific rules. Merge only the
   Vehir-related entries after reading the existing configuration and
   creating a backup.

   Do not apply lockdown while reading the integration guide. Use a
   no-lockdown configuration for the initial MCP connection, then follow the
   setup flow below.

   If there is no file for your agent, read an existing one to understand
   the pattern, then apply the same principles using your agent's own docs.

   Available guides:

   - claude-code.md — Claude Code
   - opencode.md   — OpenCode

   -- Setup flow --

   Use this sequence regardless of which integration guide you read:

   1. Check for existing config files at the relevant locations.
   2. Ask the user which scope to use, whether to apply lockdown, and how
      to merge if a config already exists. Recommend per-project or local
      scope for evaluation and testing. Recommend global scope only for
      long-term Vehir-only use.

      Lockdown is recommended for safe autonomous Vehir work, but remains
      optional. It prevents the agent from using its built-in filesystem,
      editing, search, and shell tools to bypass Vehir's capability
      boundary. This protects the content-addressed store and active
      generation from direct changes, accidental writes, manual generation
      switching, and other actions that bypass reconciliation and its
      verification gates.

      With lockdown enabled, the agent continues to work through the
      Vehir MCP tool. The restriction applies only to the agent client's
      built-in tools. It does not change Linux permissions, prevent the
      user from using their shell, or make the user's configuration
      permanently read-only. Record the user's choice, but do not apply
      lockdown yet.
   3. Create a backup of every configuration file that will be changed.
      Show the exact Vehir-related merge and obtain the user's confirmation
      before writing it.
   4. Connect the MCP server using a no-lockdown configuration. Do not apply
      lockdown before MCP — the agent needs MCP access to continue.
   5. Ask the user to restart the agent so the MCP connection takes
      effect.
   6. After restart, verify the MCP connection works. Call the Vehir
      tool with {help: null} — it must return a list of available
      tools. If it does not, stop and fix the connection before
      continuing.
   7. Once the MCP connection works, delete the downloaded archive:

      rm ~/N.tar.gz

      It has served its purpose and can always be downloaded again from
      the releases page. Do this before lockdown, while the agent still
      has its ordinary host-side file access.
   8. Recommend adding a persistent instruction for the agent. Ask the user
      which instruction file and scope to use, such as `AGENTS.md`,
      `CLAUDE.md`, or the client's equivalent. The instruction should say:

      Before working with Vehir, read
      `<chosen-location>/current/etc/DEV-GUIDE.md` and follow it.

   9. If the user requested it, apply the permission lockdown and
      pre-approve the Vehir MCP tool so it runs without prompting.
   10. Ask the user to restart the agent again so the persistent instruction
      and permission lockdown take effect.
   11. After restart, call Vehir with {help: null} again. If it fails, stop
       and ask the user to restore the backed-up client configuration or
       remove only the Vehir lockdown rules. Do not bypass the restriction
       through another host-side tool.


8. IF SOMETHING GOES WRONG

   If the installation fails partway through, do not leave the system
   in a half-configured state.

   Stop and disable the service if it was started:

   systemctl --user stop vehir-kernel
   systemctl --user disable vehir-kernel

   Remove the service file:

   rm ~/.config/systemd/user/vehir-kernel.service
   systemctl --user daemon-reload

   Do not remove the install location automatically. Report the failure and
   ask the user whether the files created during this installation attempt
   should be removed.

   Fix the problem that caused the failure, then start over from step 1.
