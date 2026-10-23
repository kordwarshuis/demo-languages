# Variants of this specification

`variants/` is the sample variants directory. The name is only a sample. In `specs.json` it is set with:

```json
"variants": {
    "directory": "variants",
    "menu_title": "Variants"
}
```

`directory` can be any folder name. Turn the feature off by removing `variants`.

## Cascade

Each subdirectory is a variant. A variant can contain another variant. Folder names are free: `de`, `plain`, `developer`, `beginner`, or any name you choose.

For every markdown file in the canonical spec, the build looks in this order:

1. The variant folder itself
2. Each parent folder, up to this directory
3. A file placed directly in this directory (shared by every variant)
4. The same file in the canonical `spec/` directory

The first file found is the one that is published. A missing translation is filled in from the parent during the build. The build does not copy that file back into the source tree. A copied source file would look like a translation, and the next build and the next AI rewrite would treat it as one.

`terms-definitions/` is not a variant. It is the folder where term files live, in the canonical spec and inside a variant. A variant must not use that folder name, or the name of any other directory that appears in `markdown_paths`.

`variant.json` is optional. It sets the title and description shown in the menu. The folder name is used when there is no title.

```text
spec/
  spec-head.md                         canonical text
  variants/                            menu, not a variant
    de/                                language first
      spec-head.md
      terms-definitions/soil.md
      plain/                           simpler text; other files come from de
        spec-body.md
    developer/                         audience first
      example-markup-in-markdown.md
      de/                              German intro; example stays from developer
        spec-head.md
      en/                              compost term only
        terms-definitions/compost.md
      nl/                              Dutch closing section
        spec-body.md
      beginner/
        spec-head.ai.md                AI draft, published
      reviewed/
        spec-head.manual.md            published
        spec-head.ai.md                kept, not published
    legal/                             another audience
      spec-head.md                     legal introduction
      de/                              replaces that introduction
        spec-head.md
      en/                              soil term only; intro stays from legal
        terms-definitions/soil.md
      nl/                              Dutch closing section
        spec-body.md
```

## Manual files and AI rewrites

In one folder, for one logical file such as `spec-head.md`:

| File | Who writes it | Published | Next AI run |
| --- | --- | --- | --- |
| `spec-head.manual.md` | A person | Yes, ahead of the others | Skipped |
| `spec-head.md` | A person | Yes, when no `.manual.md` exists | Skipped |
| `spec-head.ai.md` | AI | Yes, when no manual file exists | Replaced |

An AI rewrite is only allowed to create or replace a `.ai.md` file. It cannot write a `.md` or `.manual.md` file. When a manual file is already in the folder, that logical file is skipped, so the manual text and any sibling `.ai.md` draft stay as they are.

To keep a reviewed text, rename `spec-head.ai.md` to `spec-head.md`. To keep the draft beside the reviewed text, rename the reviewed text to `spec-head.manual.md` and leave the `.ai.md` file. The build then publishes the manual file.

See which files a new AI round would write or skip:

```bash
spec-up-t variants-plan
```

The command prints a plan and writes nothing. A program that calls a model uses `applyAiRewrites` and passes its own rewrite function. This is the whole protection: the function never opens a manual file for writing.

```javascript
const { applyAiRewrites } = require('spec-up-t/src/pipeline/variants/ai-variant-write');
const { ancestorLayers } = require('spec-up-t/src/pipeline/variants/variant-cascade');

await applyAiRewrites({
    layerDir: 'spec/variants/de',
    ancestorLayers: ancestorLayers({
        specDir: 'spec',
        variantRoot: 'spec/variants',
        relative: 'de'
    }),
    logicalPaths: ['spec-head.md', 'spec-body.md'],
    rewrite: async (logicalPath, sourceText) => {
        return translate(sourceText);
    }
});
```

`rewrite` receives the published parent text. Return `null` to skip that file. Return a string to write `<name>.ai.md`. For the sample folder `de`, `spec-head.md` is skipped because that file is already manual, and `spec-body.md` would be created as `spec-body.ai.md`.

## Build

The canonical page is `docs/index.html`. Its header links to `docs/variants/index.html`, the menu of every variant.

Each folder, including a folder with no markdown files, is built to the same path under the output directory. For this sample that is `docs/variants/de/index.html`, `docs/variants/de/plain/index.html`, `docs/variants/developer/index.html`, `docs/variants/developer/de/index.html`, `docs/variants/legal/nl/index.html`, and the same for every other folder.
