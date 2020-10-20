---
title: Validating plugin options
---

To help users [configure plugins](/docs/configuring-usage-with-plugin-options/) correctly, a plugin can optionally define a schema to enforce a type for each option. Gatsby will validate that the options users pass match the schema to help them correctly set up their site.

## How to define an options schema

To define their options schema, plugins export [`pluginOptionsSchema`](/docs/node-apis/#pluginOptionsSchema) from their `gatsby-node.js` file. This function gets passed an instance of [Joi](https://joi.dev) to define the schema with.

For example, consider the following plugin called `gatsby-plugin-console-log`:

```javascript:title=gatsby-config.js
module.exports = {
  plugins: [
    {
      resolve: `gatsby-plugin-console-log`,
      options: {
        optionA: true,
        message: "Hello world"
        optionB: false, // Optional.
      },
    },
  ],
}
```

`gatsby-plugin-console-log` can ensure users have to pass a boolean to `optionA` and a string to `message`, as well as optionally pass a boolean to `optionB`, with this `pluginOptionsSchema`:

```javascript:title=plugins/gatsby-plugin-console-log/gatsby-node.js
exports.pluginOptionsSchema = ({ Joi }) => {
  return Joi.object({
    optionA: Joi.boolean().required().description(`Enables optionA.`),
    message: Joi.string()
      .required()
      .description(`The message logged to the console.`),
    optionB: Joi.boolean().description(`Enables optionB.`),
  })
}
```

If users pass options that do not match the schema, the validation will show an error when they run `gatsby develop` and prompt them to fix their configuration.

## Unit testing an options schema

To verify that a `pluginOptionsSchema` behaves as expected, unit test it with different configurations using the [`gatsby-plugin-utils` package](https://github.com/gatsbyjs/gatsby/tree/master/packages/gatsby-plugin-utils#testpluginoptionsschema).

1. Add the `gatsby-plugin-utils` package to your site:

   ```shell
   npm install gatsby-plugin-utils
   ```

2. Use the `testPluginOptionsSchema` function exported from the package in your test file. For example, with [Jest](https://jestjs.io):

   ```javascript:title=plugins/gatsby-plugin-console/__tests__/gatsby-node.js
   // This is an example using Jest (https://jestjs.io/)
   import { testPluginOptionsSchema } from "gatsby-plugin-utils"
   import { pluginOptionsSchema } from "../gatsby-node"

   describe(`pluginOptionsSchema`, () => {
     it(`should invalidate incorrect options`, () => {
       const options = {
         optionA: undefined, // Should be a boolean
         message: 123, // Should be a string
         optionB: `not a boolean`, // Should be a boolean
       }
       const { isValid, errors } = testPluginOptionsSchema(
         pluginOptionsSchema,
         options
       )

       expect(isValid).toBe(false)
       expect(errors).toEqual([
         `"optionA" is required`,
         `"message" must be a string`,
         `"optionB" must be a boolean`,
       ])
     })

     it(`should validate correct options`, () => {
       const options = {
         optionA: false,
         message: "string",
         optionB: true,
       }
       const { isValid, errors } = testPluginOptionsSchema(
         pluginOptionsSchema,
         options
       )

       expect(isValid).toBe(true)
       expect(errors).toEqual([])
     })
   })
   ```

## Best practices for option schemas

The [Joi API documentation](https://joi.dev/api/) is a great reference to use while working on a `pluginOptionsSchema` as it shows all the available types and methods.

Here are some specific Joi best practices for `pluginOptionsSchema`s:

- [Add descriptions](#add-descriptions)
- [Set default options](#set-default-options)
- [Validate external access](#validate-external-access) where necessary
- [Add custom error messages](#add-custom-error-messages) where useful
- [Deprecate options](#deprecating-options) in a major version release rather than removing them

### Add descriptions

Make sure that every option and field has a `.description()` explaining its purpose. This is helpful for documentation as users can look at the schema and understand all the options. There might also be tooling in the future that auto-generates plugin option documentation from the schema.

### Set default options

You can use the `.default()` method to set a default value for an option. For example with `gatsby-plugin-console-log`, you can have the `message` option default to `"default message"` in case a user does not pass their own `message`:

```javascript:title=plugins/gatsby-plugin-console-log/gatsby-node.js
exports.pluginOptionsSchema = ({ Joi }) => {
  return Joi.object({
    optionA: Joi.boolean().required().description(`Enables optionA.`),
    message: Joi.string()
      .default(`default message`) // highlight-line
      .description(`The message logged to the console.`),
    optionB: Joi.boolean().description(`Enables optionB.`),
  })
}
```

Accessing `pluginOptions.message` would then log `"default message"` in all plugin APIs if the user does not supply their own value:

```javascript:title=plugins/gatsby-plugin-console-log/gatsby-node.js
exports.onPreInit = (_, pluginOptions) => {
  console.log(
    `logging: "${pluginOptions.message}" to the console` // highlight-line
  )
}
```

### Validating external access

Some plugins (particularly source plugins) query external APIs. With the `.external()` method, you can asynchronously validate that the user has access to the API, providing a better experience if they pass invalid secrets.

For example, this is how the [Contentful source plugin](/plugins/gatsby-source-contentful/) ensures the user has access to the space they are trying to query:

```javascript:title=gatsby-source-contentful/gatsby-node.js
exports.pluginOptionsSchema = ({ Joi }) => {
  return Joi.object({
    accessToken: Joi.string().required(),
    spaceId: Joi.string().required(),
    // ...more options here...
  }).external(async pluginOptions => {
    try {
      await contentful
        .createClient({
          space: pluginOptions.spaceId,
          accessToken: pluginOptions.accessToken,
        })
        .getSpace()
    } catch (err) {
      throw new Error(
        `Cannot access Contentful space "${pluginOptions.spaceId}" with the provided access token. Double check they are correct and try again!`
      )
    }
  })
}
```

### Add custom error messages

Sometimes you might want to provide more detailed error messages when validation fails for a specific field. Joi provides a [`.messages()` method](https://github.com/sideway/joi/issues/2109) which lets you override error messages for specific [error types](https://joi.dev/api/?v=17.2.1#list-of-errors).

For example, this is how we would provide a custom error message if users do not specify `optionA` for `gatsby-plugin-console-log`:

```javascript:title=plugins/gatsby-plugin-console-log/gatsby-node.js
exports.pluginOptionsSchema = ({ Joi }) => {
  return Joi.object({
    optionA: Joi.boolean()
      .required()
      .description(`Enables optionA.`)
      // highlight-start
      .messages({
        // Override the error message if the .required() call fails
        "any.required": `"optionA" needs to be specified to true or false. Get the correct value from your dashboard settings.`,
      }),
    // highlight-end
    message: Joi.string()
      .default(`default message`)
      .description(`The message logged to the console.`),
    optionB: Joi.boolean().description(`Enables optionB.`),
  })
}
```

### Deprecating options

Rather than removing options from the schema, deprecate them using the [`.forbidden()` method](https://joi.dev/api/?v=17.2.1#anyforbidden) in a major version release. Then, [add a custom error message](#add-custom-error-messages) explaining how users should upgrade the functionality using `.messages()`.

For example:

```javascript:title=plugins/gatsby-plugin-console-log/gatsby-node.js
  return Joi.object({
    optionA: Joi.boolean()
      .required()
      .description(`Enables optionA.`)
      // highlight-start
      .forbidden()
      .messages({
        // Override the error message if the .forbidden() call fails
        "any.unknown": `"optionA" is no longer supported. Use "optionB" instead by setting it to the same value you had before on "optionA".`,
      }),
      // highlight-end
    message: Joi.string()
      .default(`default message`)
      .description(`The message logged to the console.`),
    optionB: Joi.boolean().description(`Enables optionB.`),
  })
}
```

## Additional resources

- [Joi API documentation](https://joi.dev/api/)
- [`pluginOptionsSchema` for the Contentful source plugin](https://github.com/gatsbyjs/gatsby/blob/af973d4647dc14c85555a2ad8f1aff08028ee3b7/packages/gatsby-source-contentful/src/gatsby-node.js#L75-L159)