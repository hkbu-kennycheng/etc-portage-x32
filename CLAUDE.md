# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a collection of x32 (ILP32 ABI on x86-64) patches for Gentoo packages. Patches are applied automatically by Portage's `eapply_user` mechanism.

## Structure

```
patches/
  <category>/<package-version>/x32.patch    # Per-package x32 fixes
package.mask/
  <category>                               # package.mask.d-style files
package.use/
  <category>                               # package.use.d-style files
```

Patches in `patches/<category>/<package-ver>/` are auto-applied by `eapply_user` during emerge.

## Development Workflow

Test changes in the x32 container:

```bash
# Run commands in the active container
podman exec x32-openmp-test bash -c 'emerge --oneshot <package>'

# Check build logs
podman exec x32-openmp-test bash -c 'tail -20 /tmp/<log-file>'
```

The container `x32-openmp-test` is based on `kennycheng/gentoo-stage3:x32-systemd` and has:
- `/etc/portage/patches` bind-mounted to `./patches`
- `/etc/portage/package.mask` bind-mounted to `./package.mask`
- `/etc/portage/package.use` bind-mounted to `./package.use`

## Key x32 Patterns

**Ruby coroutine selection** (configure.ac):
- Gentoo ebuild forces `--with-coroutine=amd64` on x32 CHOSTs
- amd64 coroutine assembly uses 8-byte `movq` on stack pointers but x32 ILP32 has 4-byte pointers → corrupted saved context
- Fix: Override to ucontext backend after the ebuild's forced option. Use `AC_CHECK_FUNCS([getcontext swapcontext makecontext])` to set `coroutine_type=ucontext` when `ac_cv_sizeof_voidp=4`

**LLVM OpenMP** (kmp.h, etc.):
- Add `KMP_ARCH_X86_X32` to existing `KMP_ARCH_X86_64` conditionals
- May need `ld-linux-x32.so.2` in expected library deps

**General**:
- x32 has `__ILP32__` defined; pointers are 32-bit, but the x86-64 ISA (8-byte stack slots, 64-bit GPRs)
- `SIZEOF_VOIDP=4` distinguishes x32 from x86_64 (`SIZEOF_VOIDP=8`)

## Ruby-Specific Validation

After ruby rebuild, verify the coroutine backend:

```bash
# Check config.h for backend
grep COROUTINE_H /usr/include/ruby-3.*/x86_64-linux-x32/ruby/config.h

# Quick fiber test (heavy yield)
RUBYOPT="" ruby33 -e 'f = Fiber.new { |x| (1..10000).each { |i| Fiber.yield i*x }; :done }; puts (1..10000).map { |i| f.resume(i) }.size; puts f.resume(1)'
```

## Reference

The `.claude/settings.local.json` grants permissions for:
- `podman ps`, `exec`, `start`, `stop`, `rm`, `logs`, `inspect`
- `emerge -p/-pv` for package planning
- `git log`, `git remote` for version control