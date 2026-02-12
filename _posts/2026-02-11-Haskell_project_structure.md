---
title: A typical Haskell project structure
excerpt:
    Structure of a typical Haskell project and naming conventions
    from my personal point of view
tags: Haskell
---

When I develop Haskell packages, I usually use the following structure of the project directory.

## Top-level project layout

For a single-package project ALPHA it should be

    ALPHA/
    ├── stack.yaml/cabal.project
    ├── ALPHA.cabal
    └── ...other package files...see below...

For a project containing multiple packages (say, BETA and GAMMA)

    ALPHA/
    ├── stack.yaml/cabal.project
    ├── BETA/
    │   ├── BETA.cabal
    │   └── ...other BETA package files...
    └── GAMMA/
        ├── GAMMA.cabal
        └── ...other GAMMA package files...

## Package layout

Let's take a look into the GAMMA package. First, the cabal file. Nowadays, Cabal format is good enough, so we don't need `hpack` anymore.

    GAMMA/
    ├── GAMMA.cabal
    └── ...

Each component goes into a separate folder

    GAMMA/
    ├── GAMMA.cabal
    ├── lib/    -- library(ies)
    ├── exe/    -- executable(s)
    ├── test/   -- test(s)
    └── bench/  -- benchmark(s)

(Michael Snoyman likes `src` for the library and `app` for the executable. I don't get it.)

If there is only one component of each type, its files are put into its directory.

If I need to have multiple components of some type, I create one more folder level.

    GAMMA/
    ├── GAMMA.cabal
    │   ╭───────────────────────────────────────────────────────────╮
    │   │...                                                        │
    │   │                                                           │
    │   │library  -- public library                                 │
    │   │  hs-source-dirs: lib/GAMMA  -- same name as the package   │
    │   │                                                           │
    │   │library DELTA  -- internal library                         │
    │   │  hs-source-dirs: lib/DELTA                                │
    │   │                                                           │
    │   │...                                                        │
    │   ╰───────────────────────────────────────────────────────────╯
    └── lib/
        ├── GAMMA/
        │   └── GAMMA.hs    -- usually the main module and the package have
        │                   -- the same name
        └── DELTA/
            └── DELTA.hs

Same for executables, test-suites and benchmarks

    GAMMA/
    ├── GAMMA.cabal
    │   ╭───────────────────────────────╮
    │   │...                            │
    │   │                               │
    │   │executable EPSILON             │
    │   │  hs-source-dirs: exe/EPSILON  │
    │   │  main-is: Main.hs             │
    │   │                               │
    │   │executable ZETA                │
    │   │  hs-source-dirs: lib/ZETA     │
    │   │  main-is: Main.hs             │
    │   │                               │
    │   │...                            │
    │   ╰───────────────────────────────╯
    └── exe/
        ├── GAMMA/
        │   └── Main.hs
        └── DELTA/
            └── Main.hs

## Module naming

No ancient `Control`/`Data`/`Network` namespace prefixes.

Main module of an executable/test-suite/benchmark is always `Main.hs` (sometimes `Main.lhs`).

All other modules are in package namespace (`GAMMA.Text.Parser`) or in project-package one (`ALPHA.GAMMA.Text.Parser`).

## Comments?

https://t.me/haskell_and_curry/2
