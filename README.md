# jscodeshift Tricks

A collection of useful tricks for [`jscodeshift`](https://www.npmjs.com/package/jscodeshift) codemods

## Convert `defaultProps` to JavaScript Default Parameters

To help migrate off [the deprecated React API `defaultProps`](https://github.com/facebook/react/pull/16210), you can use a codemod like this to transform your `defaultProps` into JavaScript default parameters:

`migrate-defaultProps.ts`

```ts
// Convert defaultProps to default function parameters
// npx jscodeshift --parser=tsx --extensions=tsx,ts -t migrate-defaultProps.ts components/

import { API, FileInfo, Options } from 'jscodeshift';

const transform = (file: FileInfo, api: API, options: Options) => {
  const j = api.jscodeshift;
  const root = j(file.source);

  // Find the imported component name dynamically
  let styledComponentName;
  root
    .find(j.CallExpression, {
      callee: {
        name: 'styled',
      },
    })
    .forEach((path) => {
      if (
        path.value.arguments.length > 0 &&
        path.value.arguments[0].type === 'Identifier'
      ) {
        styledComponentName = path.value.arguments[0].name;
      }
    });

  // Find the styled component's template literal
  let styledTemplateLiteral;
  root.find(j.TaggedTemplateExpression).forEach((path) => {
    if (
      path.value.tag.type === 'CallExpression' &&
      path.value.tag.callee.name === 'styled' &&
      path.value.tag.arguments[0].name === styledComponentName
    ) {
      styledTemplateLiteral = path.value.quasi;
    }
  });

  // Find the property name dynamically
  let propertyName;
  root.find(j.AssignmentExpression).forEach((path) => {
    if (
      path.value.left.property &&
      path.value.left.property.name === 'defaultProps'
    ) {
      propertyName = Object.keys(
        path.value.right.properties.reduce((acc, prop) => {
          acc[prop.key.name] = true;
          return acc;
        }, {}),
      )[0];
    }
  });

  root
    .find(j.AssignmentExpression, {
      left: {
        type: 'MemberExpression',
        property: {
          name: 'defaultProps',
        },
      },
    })
    .forEach((path) => {
      const componentName = path.value.left.object.name;
      const defaultProps = path.value.right;

      // Create a new functional component with default parameters
      const newComponent = j.functionDeclaration(
        j.identifier(`Unstyled${componentName}`),
        [
          j.objectPattern([
            // Add default parameters to the new component
            ...defaultProps.properties.map((prop) => {
              const id = j.identifier(prop.key.name);
              let defaultValue;

              // Check if the value is an array expression
              if (prop.value.type === 'ArrayExpression') {
                defaultValue = j.arrayExpression(prop.value.elements);
              } else {
                // For literals, use the literal value
                defaultValue = j.literal(prop.value.value);
              }

              // Create an assignment pattern for default values
              const assignmentPattern = j.assignmentPattern(id, defaultValue);
              const property = j.property('init', id, assignmentPattern);
              property.shorthand = true; // Enable shorthand syntax
              return property;
            }),
            // Spread the rest of the properties
            j.restElement(j.identifier('props')),
          ]),
        ],
        j.blockStatement([
          j.returnStatement(
            j.jsxElement(
              j.jsxOpeningElement(
                j.jsxIdentifier(styledComponentName),
                [
                  // Spread props into the Box component
                  j.jsxSpreadAttribute(j.identifier('props')),
                  // spread the rest of the properties
                  ...defaultProps.properties.map((prop) => {
                    return j.jsxAttribute(
                      j.jsxIdentifier(prop.key.name),
                      j.jsxExpressionContainer(j.identifier(prop.key.name)),
                    );
                  }),
                ],
                true,
              ),
              null,
              [],
            ),
          ),
        ]),
      );

      // Replace the old component with the new one
      root
        .find(j.ImportDeclaration)
        .at(-1)
        .forEach((importPath) => {
          j(importPath).insertAfter(newComponent);
        });

      root
        .find(j.VariableDeclaration)
        .filter(
          (variablePath) =>
            variablePath.value.declarations[0].id.name === componentName,
        )
        .forEach((variablePath) => {
          variablePath.value.declarations[0].init = j.taggedTemplateExpression(
            j.callExpression(j.identifier('styled'), [
              j.identifier('Unstyled' + componentName),
            ]),
            styledTemplateLiteral,
          );
        });

      j(path).remove();
    });

  return root.toSource({ quote: 'single' });
};

export default transform;
```


Input:

```tsx
import styled from '@emotion/styled';
import { Box } from 'rebass';

const Container = styled(Box)`
  margin-left: auto;
  margin-right: auto;
  max-width: 1310px;
`;

Container.defaultProps = {
  px: 3,
};

export default Container;
```

Output:

```tsx
import styled from '@emotion/styled';
import { Box } from 'rebass';

function UnstyledContainer(
  {
    px = 3,
    ...props
  }
) {
  return <Box {...props} px={px} />;
}

const Container = styled(UnstyledContainer)`
  margin-left: auto;
  margin-right: auto;
  max-width: 1310px;
`;

export default Container;
```

Source: originally posted in this Emotion issue: https://github.com/emotion-js/emotion/issues/2573#issuecomment-1994603166
