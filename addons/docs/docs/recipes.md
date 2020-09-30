<h1>Storybook Docs Recipes</h1>

[Storybook Docs](../README.md) consists of two basic mechanisms, [DocsPage](docspage.md) and [MDX](mdx.md). But how should you use them in your project?

- [Component Story Format (CSF) with DocsPage](#component-story-format-csf-with-docspage)
- [Pure MDX Stories](#pure-mdx-stories)
- [Mixed CSF / MDX Stories](#mixed-csf--mdx-stories)
- [CSF Stories with MDX Docs](#csf-stories-with-mdx-docs)
- [CSF Stories with arbitrary MDX](#csf-stories-with-arbitrary-mdx)
- [Mixing storiesOf with CSF/MDX](#mixing-storiesof-with-csfmdx)
- [Migrating from notes/info addons](#migrating-from-notesinfo-addons)
- [Exporting documentation](#exporting-documentation)
- [Disabling docs stories](#disabling-docs-stories)
  - [DocsPage](#docspage)
  - [MDX Stories](#mdx-stories)
- [Controlling a story's view mode](#controlling-a-storys-view-mode)
- [More resources](#more-resources)

## Component Story Format (CSF) with DocsPage

Storybook's [Component Story Format (CSF)](https://medium.com/storybookjs/component-story-format-66f4c32366df) is a convenient, portable way to write stories. [DocsPage](docspage.md) is a convenient, zero-config way to get rich docs for CSF stories. Using these together is a primary use case for Storybook Docs.

If you want to intersperse longform documentation in your Storybook, for example to include an introductory page at the beginning of your storybook with an explanation of your design system and installation instructions, [Documentation-only MDX](mdx.md#documentation-only-mdx) is a good way to achieve this.

## Pure MDX Stories

[MDX](mdx.md) is an alternative syntax to CSF that allows you to co-locate your stories and your documentation. Everything you can do in CSF, you can do in MDX. And if you're consuming it in [Webpack](https://webpack.js.org/), it exposes an _identical_ interface, so the two files are interchangeable. Some teams will choose to write all of their Storybook in MDX and never look back.

## Mixed CSF / MDX Stories

Can't decide between CSF and MDX? In transition? Or have did you find that each format has its own use? There's nothing stopping you from keeping some of your stories in CSF and some in MDX. And if you want to migrate one way or another, the [csf-to-mdx and mdx-to-csf codemod migrations](https://github.com/storybookjs/storybook/blob/next/lib/codemod/README.md) can help.

The only limitation is that your exported titles (CSF: `default.title`, MDX `Meta.title`) should be unique across files. Loading will fail if there are duplicate titles.

## CSF Stories with MDX Docs

Perhaps you want to write your stories in CSF, but document them in MDX? Here's how to do that:

**Button.stories.js**

```js
import React from 'react';
import { Button } from './Button';

export default {
  title: 'Demo/Button',
  component: Button,
  includeStories: [], // or don't load this file at all
};

export const basic = () => <Button>Basic</Button>;
basic.story = {
  parameters: { foo: 'bar' },
};
```

**Button.stories.mdx**

```md
import { Meta, Story } from '@storybook/addon-docs/blocks';
import \* as stories from './Button.stories.js';
import { SomeComponent } from 'path/to/SomeComponent';

<Meta title="Demo/Button" component={Button} />

# Button

I can define a story with the function imported from CSF:

<Story name="basic">{stories.basic()}</Story>

And I can also embed arbitrary markdown & JSX in this file.

<SomeComponent prop1="val1" />
```

What's happening here:

- Your stories are defined in CSF, but because of `includeStories: []`, they are not actually added to Storybook.
- The MDX file is simply importing stories as functions in the MDX, and other aspects of the CSF file, such as decorators, parameters, and any other metadata should be applied as needed in the MDX from the import.

## CSF Stories with arbitrary MDX

We recommend [MDX Docs](#csf-stories-with-mdx-docs) as the most ergonomic way to annotate CSF stories with MDX. There's also a second option if you want to annotate your CSF with arbitrary markdown:

**Button.mdx**

```md
import { Story } from '@storybook/addon-docs/blocks';
import { SomeComponent } from 'somewhere';

# Button

I can embed a story (but not define one, since this file should not contain a `Meta`):

<Story id="some--id" />

And I can also embed arbitrary markdown & JSX in this file.

<SomeComponent prop1="val1" />
```

**Button.stories.js**

```js
import React from 'react';
import { Button } from './Button';
import mdx from './Button.mdx';
export default {
  title: 'Demo/Button',
  parameters: {
    docs: {
      page: mdx,
    },
  },
  component: Button,
};
export const basic = () => <Button>Basic</Button>;
```

Note that in contrast to other examples, the MDX file suffix is `.mdx` rather than `.stories.mdx`. This key difference means that the file will be loaded with the default MDX loader rather than Storybook's CSF loader, which has several implications:

1. You shouldn't provide a `Meta` declaration.
2. You can refer to existing stories (i.e. `<Story id="...">`) but cannot define new stories (i.e. `<Story name="...">`).
3. The documentation gets exported as the default export (MDX default) rather than as a parameter hanging off the default export (CSF).

## Mixing storiesOf with CSF/MDX

You might have a collection of stories using the `storiesOf` API and want to add CSF/MDX piecemeal. Or you might have certain stories that are only possible with the `storiesOf` API (e.g. dynamically generated ones)

So how do you mix these two types? The first argument to `configure` can be a `require.context "req"`, an array of `req's`, or a `loader function`. The loader function should either return null or an array of module exports that include the default export. The default export is used by `configure` to load CSF/MDX files.

So here's a naive implementation of a loader function that assumes that none of your `storiesOf` files contains a default export, and filters out those exports:

```js
const loadFn = () => {
  const req = require.context('../src', true, /\.stories\.js$/);
  return req
    .keys()
    .map(fname => req(fname))
    .filter(exp => !!exp.default);
};

configure(loadFn, module);
```

We could have baked this heuristic into Storybook, but we can't assume that your `storiesOf` files don't have default exports. If they do, you can filter them some other way (e.g. by file name).

If you don't filter out those files, you'll see the following error:

> "Loader function passed to 'configure' should return void or an array of module exports that all contain a 'default' export"

We made this error explicit to make sure you know what you're doing when you mix `storiesOf` and CSF/MDX.

## Migrating from notes/info addons

If you're currently using the notes/info addons, you can upgrade to DocsPage by providing a custom `docs.extractComponentDescription` parameter. There are different ways to use each addon, so you can adapt this recipe according to your use case.

Suppose you've added a `notes` parameter to each component in your library, containing markdown text, and you want that to show up at the top of the page in the `Description` slot. You could do that by adding the following snippet to `.storybook/preview.js`:

```js
import { addParameters } from '@storybook/client-api';

addParameters({
  docs: {
    extractComponentDescription: (component, { notes }) => {
      if (notes) {
        return typeof notes === 'string' ? notes : notes.markdown || notes.text;
      }
      return null;
    },
  },
});
```

The default `extractComponentDescription` provided by the docs preset extracts JSDoc code comments from the component source, and ignores the second argument, which is the story parameters of the currently-selected story. In contrast, the code snippet above ignores the comment and uses the notes parameter for that story.

## Exporting documentation

> ⚠️ The `--docs` flag is an experimental feature in Storybook 5.2. The behavior may change in 5.3 outside of the normal semver rules. Be forewarned!

The Storybook UI is a workshop for developing components in isolation. Storybook Docs is a showcase for documenting your components. During component/docs development, it’s useful to see both of these modes side by side. But when you export your static storybook, you might want to export the docs to reduce clutter.

To address this, we’ve added a CLI flag to only export the docs. This flag is also available in dev mode:

```sh
yarn build-storybook --docs
```

## Disabling docs stories

There are two cases where a user might wish to exclude stories from their documentation pages:

### DocsPage

User defines stories in CSF and renders docs using DocsPage, but wishes to exclude some fo the stories from the DocsPage to reduce noise on the page.

```js
export const foo = () => <Button>foo</Button>;
foo.story = { parameters: { docs: { disable: true } } };
```

### MDX Stories

User writes documentation & stories side-by-side in a single MDX file, and wants those stories to show up in the canvas but not in the docs themselves. They want something similar to the recipe "CSF stories with MDX docs" but want to do everything in MDX:

```js
<Story name="foo" parameters={{ docs: { disable: true }} >
  <Button>foo</Button>
</Story>
```

## Controlling a story's view mode

Storybook's default story navigation behavior is to preserve the existing view mode. In other words, if a user is viewing a story in "docs" mode, and clicks on another story, they will navigate to the other story in "docs" mode. If they are viewing a story in "story" mode (i.e. "canvas" in the UI) they will navigate to another story in "story" mode (with the exception of "docs-only" pages, which are always shown in "docs" mode).

Based on user feedback, it's also possible to control the view mode for an individual story using the `viewMode` story parameter. In the following example, the nav link will always set the view mode to story:

```js
export const Foo = () => <Component />;
Foo.story = {
  parameters: {
    // reset the view mode to "story" whenever the user navigates to this story
    viewMode: 'story',
  },
};
```

This can also be applied globally in `preview.js`:

```js
// always reset the view mode to "docs" whenever the user navigates
addParameters({
  viewMode: 'docs',
});
```

## More resources

- References: [README](../README.md) / [DocsPage](docspage.md) / [MDX](mdx.md) / [FAQ](faq.md) / [Recipes](recipes.md) / [Theming](theming.md)
- Framework-specific docs: [React](../react/README.md) / [Vue](../vue/README.md) / [Angular](../angular/README.md) / [Web components](../web-components/README.md) / [Ember](../ember/README.md)
- Announcements: [Vision](https://medium.com/storybookjs/storybook-docs-sneak-peak-5be78445094a) / [DocsPage](https://medium.com/storybookjs/storybook-docspage-e185bc3622bf) / [MDX](https://medium.com/storybookjs/rich-docs-with-storybook-mdx-61bc145ae7bc) / [Framework support](https://medium.com/storybookjs/storybook-docs-for-new-frameworks-b1f6090ee0ea)
- Example: [Storybook Design System](https://github.com/storybookjs/design-system)