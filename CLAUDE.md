# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

timetracker is a Vue 3 + TypeScript application built with Vite.

## Commands

- `npm run dev` - Start development server with HMR
- `npm run build` - Type-check with vue-tsc then build for production
- `npm run preview` - Preview the production build locally

## Architecture

- **Entry point:** `src/main.ts` mounts the Vue app to `#app` in `index.html`
- **Root component:** `src/App.vue`
- **Components:** `src/components/`
- **Global styles:** `src/style.css` (imported in main.ts); components use `<style scoped>`
- Components use Vue 3 `<script setup>` syntax with Composition API
- No router or state management library is configured yet

## TypeScript

Strict mode is enabled with additional checks: `noUnusedLocals`, `noUnusedParameters`, `erasableSyntaxOnly`, `noFallthroughCasesInSwitch`, `noUncheckedSideEffectImports`.
