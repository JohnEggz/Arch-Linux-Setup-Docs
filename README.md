# arch-setup

Personal notes on building and configuring an Arch Linux laptop — a chronological record of
**what** was done, **why**, and **how**, verified against the live system.

📄 **Main document: [arch-setup.md](arch-setup.md)**

Covers, in order: install → disk/LUKS/btrfs → boot (Limine + UKI + plymouth) → network/firewall →
zram/power → audio → bluetooth → printing → graphics → zsh → fonts → ghostty → Hyprland/UWSM →
portals → apps, then a cleanup log, known issues, and a todo list.

## Reading it

The document uses grep-friendly tags:

```sh
grep -n 'status/todo'    arch-setup.md   # everything still to do
grep -n 'status/done'    arch-setup.md   # completed
grep -n '#inferred'      arch-setup.md   # reconstructed rationale
grep -n '#verify'        arch-setup.md   # needs root/manual confirmation
```

## Notes

- This is a personal reference, not a guide. Hardware-specific values (UUIDs, hostname) are left
  as-is; they are identifiers, not secrets.
- No credentials, keys, tokens, or passphrases are stored here.
