# Build-variants

Single function to create, manage, compose variants, for any CSS-in-JS libraries.


## Installation

```bash
npm install build-variants
```

## How to use

### Step 1 - build-variants configuration

To be able to use build-variants with your CSS-in-JS library and get valid typings
for your CSS/styles, you need to specify the type of the CSS object.

To do so, you can create a simple function that expose `newBuildVariants` with
the CSS interface that you want to use.

For example with styled-components:

```ts
import { newBuildVariants } from 'build-variants'
import { CSSObject } from 'styled-components'

/**
 * Create a BuildVariants instance, typed to use styled-components's `CSSObject`s.
 */
export function buildVariants<TProps extends object>(props: TProps) {
  return newBuildVariants<TProps, CSSObject>(props)
}
```

Note: If your library doesn't expose typings or if you are doing raw CSS only with
React, you can use `React.CSSProperties` for example.

You can even use you own CSS declaration if you want to use build-variants in a
totally different context:

```ts
import { newBuildVariants } from 'build-variants'

// build-variants typings will only tolerate CSS with color and background properties!
interface IMyStyles {
  color: string
  background: string
}

/**
 * Create a BuildVariants instance, typed to use styled-components's `CSSObject`s.
 */
export function buildVariants<TProps extends object>(props: TProps) {
  return newBuildVariants<TProps, Partial<IMyStyles>>(props)
}
```


### Step 2 - Build your variants!


```tsx
import { buildVariants } from 'path/to/buildVariants'

// Use here styled-components but you can used any CSS-in-JS library you want
import { styled } from 'styled-components'

// Define the interface of the props used by your component
interface Props {
  // Define a 'private' property for the button font color
  _color?: 'default' | 'primary' | 'secondary'

  // Define a 'private' property for the button background
  _background?: 'default' | 'primary' | 'secondary'

  // Define a 'private' property for font variants.
  // This is an array, meaning that you can apply several values at once.
  _font?: Array<'default' | 'bold' | 'italic'>,

  type?: 'default' | 'primary' | 'secondary'
}

// Style a div component by using styled-components here.
const Div = styled.div<Props>props => {
  // Get a new instance of build-variant.
  // Note that we use here `buildVariants()` defined in step 1 to be able to write
  // CSS styles as styled-components' CSS objects.
  return buildVariants(props)
    // Add some CSS.
    .css({
      background: 'white'
    })

    // Add more CSS.
    // You can add as many CSS blocks as you want.
    .css({
      '> button': {
        all: 'unset'
      }
    })

    // Implement CSS for each case of the color variant.
    // Everything is typed checked here. You have to implement all cases of the
    // union value.
    // Note that because _color is optional, we have to default on a default value,
    // here 'default', which is the first value of the union.
    .variant('_color', props._color || 'default', {
      default: {
        // No color for the default case, it will be inherited from a parent
      },

      primary: {
        color: 'white'
      },

      secondary: {
        color: 'black'
      }
    })

    // Same thing with the background variant
    .variant('_background', props._background || 'default', {
      default: {
        // No background override.
        // As we have define a background in the first CSS block, we should have
        // a background: white at the end.
      },

      primary: {
        background: 'blue'
      },

      secondary: {
        background: 'white'
      }
    })

    // Same thing with the font variant.
    // Notice that we use `variants` to manipulate an array of unions.
    .variants('_font', props._font || [], {
      default: {
        // Inherits from the parent
      },

      bold: {
        fontWeight: 'bold'
      },

      italic: {
        fontStyle: 'italic'
      }
    })

    // Now, compose with your 'private' variants
    .compoundVariant('type', props.type || 'default', {
      // When composing, we get a new instance of the builder to get existing
      // private variants definitions.
      // Final `end()` function merges all CSS definitions get from the composition.
      // Here we don't want to style the default case, so we directly call the
      // end function.
      default: builder.end()

      // Here we define the type=primary variant from existing color, background
      // and font variants.
      // In this example, we use two definitions for the font variant. So the font
      // will be bold and italic.
      primary: builder
        .get('_color', 'primary')
        .get('_background', 'primary')
        .get('_font', ['bold', 'italic'])
        .end(),

      // In the same way, compose to define type=secondary variant.
      secondary: builder
        .get('_color', 'secondary')
        .get('_background', 'secondary')
        .get('_font', ['bold', 'italic'])
        .end()
    })

    // You can also compose with an array of compoundVariant by using compoundVariants:
    // .compoundVariants('types', props.types || [], {
    //   ...
    // })

    // You can conditionate any CSS or variant definition by using `if()` block.
    .if(
      // Implement the predicate function here to have a pink color in your button
      true    // OR `props.variants?.include('fancy') === true,
      builder => {
        return builder
          .css({
            color: 'pink'
          })

          // The nice trick with `if` is that this variant can be automatically
          // "skipped" in compound variants if
          .variant('_color', props._color || 'default', {
             // ...
          }

          .end()
      }
    }

    // The nice trick with `if` is that variants will be automatically "skipped"
    // from compound variants when being disabled.
    // So your compount variants dont need to have any logic to apply or not
    // internal variants, you can conditionnate them outside the composition.
    .if(
      false,
      builder => {
        return builder
          .variant('_color', props._color || 'default', {
             // ...
          }

          .end()
      }
    }

    // And finally, for various reason, you want to override the color defined
    // previously.
    // So you can use the `weight` option to ponderate your CSS definition(s).
    // Weight is also available for variant(s), compoundVariant(s) and if blocks.
    css({
      color: 'lime'
    }, {
      // By default, weight is 0 when not set. So here, color:lime will be applied last,
      // overriding previous definitions blocks.
      weight: 10
    })

    // If you have some issues and unexpected CSS applied, you may want debug things
    // so you can use `debug()` function that will log props, variants, CSS parts and
    // final merged CSS object.
    .debug()

    // Finally, merge all CSS definitions and variants.
    // End function will return a CSS object.
    .end()
})

// Create a component and render a "primary" button.
function ButtonComponent() {
  return (
    <Div type="primary">
      <button>Button</button>
    </Div>
  )
}

/* CSS will be:

{
  // defined in the first block
  '> button': {
    all: 'unset'
  },

  // get from the primary background variant, it will override the background set in the second CSS block
  background: 'blue',

  // get from the primary font variant, two variants applyed at the same time
  fontWeight: 'bold',
  fontStyle: 'italic'

  // finally, the color will be lime, because the weight option overrides the color defined in the primary variant.
  color: 'lime'
}

*/
```

Have fun building variants! :)
