# base

## Project setup

```
# yarn
yarn

# npm
npm install

# pnpm
pnpm install
```

### Compiles and hot-reloads for development

```
# yarn
yarn dev

# npm
npm run dev

# pnpm
pnpm dev
```

### Compiles and minifies for production

```
# yarn
yarn build

# npm
npm run build

# pnpm
pnpm build
```

### Lints and fixes files

```
# yarn
yarn lint

# npm
npm run lint

# pnpm
pnpm lint
```

### Customize configuration

See [Configuration Reference](https://vitejs.dev/config/).

### TODO:

- [x] [multi-touch pan/zoom support](https://konvajs.org/docs/sandbox/Multi-touch_Scale_Stage.html)
- [x] fix broken canvas pan when transformer is activated
- [ ] [grid background](https://konvajs.org/docs/sandbox/Multi-touch_Scale_Stage.html), maybe add the grid in a new fastlayer under stage or directly draw under the acvite layer
- [ ] [select on canvas](https://konvajs.org/docs/select_and_transform/Basic_demo.html)
- [ ] menu to change vuetify theme, maybe material or customizable, at least a dark and a light mode
- [ ] export/import all canvas(json/whole image?)
- [ ] copy image into clipboard
- [ ] save to /restore from indexedDB
- [ ] menu to select all / clear / paste image
- [ ] right click menu to reset transform/delete/copy image
- [ ] undo/redo
- [ ] automatic image arrange function
- [ ] support for avif compression

### Maybe TODO:
- [ ] indicator of RAM/storage usage
- [ ] tauri packing