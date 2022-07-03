---
{"dg-publish":true,"dg-permalink":"ts-eslint/plugin/custom-rule","permalink":"/ts-eslint/plugin/custom-rule/","dgHomeLink":true,"dgPassFrontmatter":false}
---


> [!important]
> 您应该熟悉[ESLint's developer guide](https://eslint.org/docs/developer-guide) 和 [Development > Architecture](./architecture/asts) 在编写自定义规则之前.

只要您在 ESLint 配置中使用`@typescript-eslint/parser`作为`parser`, 自定义 ESLint 规则通常以相同的方式为 JavaScript 和 TypeScript 代码工作。
自定义规则编写的主要三个变化是:

- [Utils Package](#utils-package): 我们建议使用`@typescript-eslint/utils`来创建自定义规则
- [AST Extensions](#ast-extensions): 在规则选择器中定位特定于 TypeScript 的语法
- [Typed Rules](#typed-rules): 使用 TypeScript 类型检查器辅助规则逻辑

## Utils Package

 `@typescript-eslint/utils` 可以作为`eslint` 的替代包，它导出所有相同的对象和类型，同时支持 `typescript-eslint`.
它还导出大多数自定义 typescript-eslint 规则倾向于使用的常见实用程序函数和常量。

> [!warning]
> `@types/eslint`类型基于`@types/estree`，并且不识别typescript-eslint节点和属性。
> 在 TypeScript 中编写自定义 typescript-eslint 规则时，通常不需要从 `eslint` 导入。

### RuleCreator

创建自定义 ESLint 规则的推荐方法是使用 `ESLintUtils.RuleCreator` 函数，该函数由`@typescript-eslint/utils`导出。

它接收一个函数，该函数将规则名称转换为其文档 URL。然后返回一个接收规则模块对象的函数。

`RuleCreator`将从提供的`meta.messages`对象推断出允许使用的message ID。(就是在调用report时能智能提示)


下面这个规则作用是禁止以小写字母开头的函数声明:
```ts
import { ESLintUtils } from '@typescript-eslint/utils';

const createRule = ESLintUtils.RuleCreator(
  name => `https://example.com/rule/${name}`,
);

// Type: RuleModule<"uppercase", ...>
export const rule = createRule({
  create(context) {
    return {
      FunctionDeclaration(node) {
        if (node.id != null) {
          if (/^[a-z]/.test(node.id.name)) {
            context.report({
              messageId: 'uppercase',
              node: node.id,
            });
          }
        }
      },
    };
  },
  name: 'uppercase-first-declarations',
  meta: {
    docs: {
      description:
        'Function declaration names should start with an upper-case letter.',
      recommended: 'warn',
    },
    messages: {
      uppercase: 'Start this name with an upper-case letter.',
    },
    type: 'suggestion',
    schema: [],
  },
  defaultOptions: [],
});
```


`RuleCreator`规则创建者函数返回类型为`RuleModule`规则对象，`RuleModule`由`@typescript-eslint/utils`导出。
它允许指定泛型:

- `MessageIds`: 用于report的字符串文本消息 ID 的并集
- `Options`: 用户可以为规则配置的选项（默认情况下为`[]`）

If the rule is able to take in rule options, declare them as a tuple type containing a single object of rule options:


如果规则支持规则选项，请将其声明为包含规则选项的单个对象的元组类型:

```ts
import { ESLintUtils } from '@typescript-eslint/utils';

type MessageIds = 'lowercase' | 'uppercase';

type Options = [
  {
    preferredCase?: 'lower' | 'upper';
  },
];

// Type: RuleModule<MessageIds, Options, ...>
export const rule = createRule<Options, MessageIds>({
  // ...
});
```

### Undocumented Rules

尽管通常不建议在没有文档的情况下创建自定义规则，但如果确定要执行此操作，则可以使用`ESLintUtils.RuleCreator.withoutDocs`函数直接创建规则。

它跟上面的`createRule`有相同的类型，但不强制需要文档URL

```ts
import { ESLintUtils } from '@typescript-eslint/utils';

export const rule = ESLintUtils.RuleCreator.withoutDocs({
  create(context) {
    // ...
  },
  meta: {
    // ...
  },
});
```
(译注: 这个方法创建规则时不允许传入`name`, 但是规则最后会放到一个大对象里, 此时的key就是name)
> [!warning]
> 建议任何自定义 ESLint 规则都包含描述性错误消息和指向信息性文档的链接。

## AST Extensions


`@typescript-eslint/estree` 为 TypeScript 语法创建 AST 节点，其名称以 `TS` 开头，例如 `TSInterfaceDelaration` 和 `TSTypeAnnotation`。

这些节点的处理方式与任何其他 AST 节点相同。
您可以在规则选择器中查询它们。


此版本的规则改为禁止以小写字母开头的接口声明名称:
```ts
import { ESLintUtils } from '@typescript-eslint/utils';

export const rule = createRule({
  create(context) {
    return {
      TSInterfaceDeclaration(node) {
        if (/^[a-z]/.test(node.id.name)) {
          // ...
        }
      },
    };
  },
  // ...
});
```

### Node Types

TypeScript types for nodes exist in a `TSESTree` namespace exported by `@typescript-eslint/utils`.
The above rule body could be better written in TypeScript with a type annotation on the `node`:

An `AST_NODE_TYPES` enum is exported as well to hold the values for AST node `type` properties.
`TSESTree.Node` is available as union type that uses its `type` member as a discriminant.

For example, checking `node.type` can narrow down the type of the `node`:

```ts
import { AST_NODE_TYPES, TSESTree } from '@typescript-eslint/utils';

export function describeNode(node: TSESTree.Node): string {
  switch (node.type) {
    case AST_NODE_TYPES.ArrayExpression:
      return `Array containing ${node.elements.map(describeNode).join(', ')}`;

    case AST_NODE_TYPES.Literal:
      return `Literal value ${node.raw}`;

    default:
      return '🤷';
  }
}
```

### Explicit Node Types

Rule queries that use more features of [esquery](https://github.com/estools/esquery) such as targeting multiple node types may not be able to infer the type of the `node`.
In that case, it is best to add an explicit type declaration.

This rule snippet targets name nodes of both function and interface declarations:

```ts
import { AST_NODE_TYPES, ESLintUtils } from '@typescript-eslint/utils';

export const rule = createRule({
  create(context) {
    return {
      'FunctionDeclaration, TSInterfaceDeclaration'(
        node:
          | AST_NODE_TYPES.FunctionDeclaration
          | AST_NODE_TYPES.TSInterfaceDeclaration,
      ) {
        if (/^[a-z]/.test(node.id.name)) {
          // ...
        }
      },
    };
  },
  // ...
});
```

## Type Checking

:::tip
Read TypeScript's [Compiler APIs > Using the Type Checker](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API#using-the-type-checker) section for how to use a program's type checker.
:::

The biggest addition typescript-eslint brings to ESLint rules is the ability to use TypeScript's type checker APIs.

`@typescript-eslint/utils` exports an `ESLintUtils` namespace containing a `getParserServices` function that takes in an ESLint context and returns a `parserServices` object.

That `parserServices` object contains:

- `program`: A full TypeScript `ts.Program` object
- `esTreeNodeToTSNodeMap`: Map of `@typescript-eslint/estree` `TSESTree.Node` nodes to their TypeScript `ts.Node` equivalents
- `tsNodeToESTreeNodeMap`: Map of TypeScript `ts.Node` nodes to their `@typescript-eslint/estree` `TSESTree.Node` equivalents

By mapping from ESTree nodes to TypeScript nodes and retrieving the TypeScript program from the parser services, rules are able to ask TypeScript for full type information on those nodes.

This rule bans for-of looping over an enum by using the type-checker via typescript-eslint and TypeScript APIs:

```ts
import { ESLintUtils } from '@typescript-eslint/utils';
import * as ts from 'typescript';
import * as tsutils from 'tsutils';

export const rule: eslint.Rule.RuleModule = {
  create(context) {
    return {
      ForOfStatement(node) {
        // 1. Grab the TypeScript program from parser services
        const parserServices = ESLintUtils.getParserServices(context);
        const checker = parserServices.program.getTypeChecker();

        // 2. Find the backing TS node for the ES node, then that TS type
        const originalNode = parserServices.esTreeNodeToTSNodeMap.get(
          node.right,
        );
        const nodeType = checker.getTypeAtLocation(originalNode);

        // 3. Check the TS node type using the TypeScript APIs
        if (tsutils.isTypeFlagSet(nodeType, ts.TypeFlags.EnumLike)) {
          context.report({
            messageId: 'loopOverEnum',
            node: node.right,
          });
        }
      },
    };
  },
  meta: {
    docs: {
      category: 'Best Practices',
      description: 'Avoid looping over enums.',
    },
    messages: {
      loopOverEnum: 'Do not loop over enums.',
    },
    type: 'suggestion',
    schema: [],
  },
};
```

## Testing

`@typescript-eslint/utils` exports a `RuleTester` with a similar API to the built-in [ESLint `RuleTester`](https://eslint.org/docs/developer-guide/nodejs-api#ruletester).
It should be provided with the same `parser` and `parserOptions` you would use in your ESLint configuration.

### Testing Untyped Rules

For rules that don't need type information, passing just the `parser` will do:

```ts
import { ESLintUtils } from '@typescript-eslint/utils';
import rule from './my-rule';

const ruleTester = new ESLintUtils.RuleTester({
  parser: '@typescript-eslint/parser',
});

ruleTester.run('my-rule', rule, {
  valid: [
    /* ... */
  ],
  invalid: [
    /* ... */
  ],
});
```

### Testing Typed Rules

For rules that do need type information, `parserOptions` must be passed in as well.
Tests must have at least an absolute `tsconfigRootDir` path provided as well as a relative `project` path from that directory:

```ts
import { ESLintUtils } from '@typescript-eslint/utils';
import rule from './my-typed-rule';

const ruleTester = new ESLintUtils.RuleTester({
  parser: '@typescript-eslint/parser',
  parserOptions: {
    project: './tsconfig.json',
    tsconfigRootDir: __dirname,
  },
});

ruleTester.run('my-typed-rule', rule, {
  valid: [
    /* ... */
  ],
  invalid: [
    /* ... */
  ],
});
```

:::note
For now, `ESLintUtils.RuleTester` requires the following physical files be present on disk for typed rules:

- `tsconfig.json`: tsconfig used as the test "project"
- One of the following two files:
  - `file.ts`: blank test file used for normal TS tests
  - `file.tsx`: blank test file used for tests with `parserOptions: { ecmaFeatures: { jsx: true } }`

:::