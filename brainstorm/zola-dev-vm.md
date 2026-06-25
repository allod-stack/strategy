# Zola Dev VM

A minimal dev VM dedicated to Zola static site development. First real-world
consumer of the `profiles` framework -- proves a single-purpose VM can be
built with just one extra package on top of the base set.

## Key Ideas

- `zola` is the only package beyond the standard base set
- System-level config is empty; all customization in home-manager
- Dev preview via `zola serve` + SSH tunnel from host
- Production build via `zola build` with deployment to self-hosted forge
  infrastructure as a follow-on task

## Status

**Not implemented.** The minimal packaging story and dev-to-production pipeline
pattern are reusable ideas for future single-purpose dev VMs.
