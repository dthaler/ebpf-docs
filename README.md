# eBPF Standard Documentation

This repository is a working draft of standard eBPF documentation
to be published by the eBPF Foundation in PDF format.

The authoritative source from which it is built is expected to be
in the Linux kernel.org repository, but not be Linux specific.

A GitHub mirror should be used, as is presently done for libbpf and
bpftool, so that other platforms and tools can easily use it.
As such, the documentation uses the subset of RST that GitHub
renders correctly.

The documentation can be IETF RFC style MUST/SHOULD/MAY language
if desired.  It does not currently do so.

## Questions

1. Any objections to the github mirror approach?

2. Should we include or exclude Linux implementation notes
   and Clang implementation notes?

3. How do we handle the legacy packet instructions?

4. How do we handle the wide instructions that reference
   map fd, map indices, BTF ids, and BPF callbacks?
