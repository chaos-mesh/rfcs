# OpenAPI to TypeScript API Client and Forms

## Summary

Use the OpenAPI specification to generate a TypeScript API client and forms.

## Motivation

Ref: https://github.com/chaos-mesh/chaos-mesh/issues/2615.

Currently, the chaos types are split between the front-end and back-end, and we
can‘t reuse types that have already been defined.

You can find these **extra** type definitions in https://github.com/chaos-mesh/chaos-mesh/blob/release-2.1/ui/app/src/components/NewExperimentNext/data/types.ts.

This leads to several problems:

1. The front-end needs to be manually synchronized when there is an update to the
   type definition.
2. As `1` continues to emerge, there are more and more things to maintain manually.

Considering our existing maintenance cost and past experience, we are not able
to synchronize the corresponding descriptions to the front-end in a timely manner.

This also works against those who want to contribute to Chaos Mesh, and it would
be great if only modifying the API will synchronize the changes to the UI.

So the best solution for now is to automate the generation of the schemas needed
for the front-end.

## Detailed design

I will split the details into two parts:

- The first part is the generation of the TypeScript schemas.
- The second part is how to use the generated files to produce a forms skeleton.

### Generate TypeScript Schemas

Luckily, there has serval tools can help us to generate the schemas, all of them
has pros and cons, finally I choose the
[OpenAPITools/openapi-generator](https://github.com/OpenAPITools/openapi-generator),
because:

- Pros
  - It's official.
  - It's a very popular tool and has a huge community.
- Cons
  - The nodejs package is just a wrapper of jar, so JRE is needed.

Other tries:

- [openapi-typescript](https://www.npmjs.com/package/openapi-typescript)
- [openapi-typescript-codegen](https://www.npmjs.com/package/openapi-typescript-codegen)

Both of these tools also have it's pros and cons, as opposite of above described.

Then we can use create a new package to handle the generation:

```sh
mkdir -p ui/packages/openapi
cd ui/package/openapi
yarn init
```

Install dependencies and add the `generate` script:

```json
{
  "scripts": {
    "generate": "export TS_POST_PROCESS_FILE='../../node_modules/.bin/prettier --write'; openapi-generator-cli generate -c openapiconfig.json -i ../../../pkg/dashboard/swaggerdocs/swagger.yaml -g typescript-axios -o ../../app/src/openapi --enable-post-process-file"
  }
}
```

This script will output the files into `ui/app/src/openapi` directory directly to
to avoid additional compilation.

### Use TypeScript Compiler API to Generate Forms

To generate forms, we have think about these things before we write the code:

- Construct the interface of a form field.
- How to handle dependencies between fields?
- How to find shared fields?
- How to use the generated schemas to construct the interface?

For the first item, a form field should normally contain at least:

- type
- label
- value
- description

Convert to a JSON:

```json
{
  "field: "text",
  "label": "Name",
  "value": "",
  "helperText": "Fill your name",
}
```

The types we currently have are:

```ts
type FieldType =
  | "text"
  | "textarea"
  | "number"
  | "select"
  | "label"
  | "autocomplete";
```

Most of them are inherited from the HTML Input types, except `label` and `autocomplete`.
The `label` is represented as `string[]`. The `autocomplete` can be ignored for
now because all generated fields won't use it.

Done. Next we need to think about how to handle dependencies between fields.

#### Dependencies between fields

Mostly, we use an `action` field to distinguish exactly what we want a chaos to
do with the injection. So I'll start from here, but how we can find different actions?
The key problem is OpenAPI generator can't convert Go `type alias` to TS `enums`,
for example:

```go
const (
        Ec2Stop AWSChaosAction = "ec2-stop"
        Ec2Restart AWSChaosAction = "ec2-restart"
        DetachVolume AWSChaosAction = "detach-volume"
)
```

We expect these can covert to:

```ts
enum AWSChaosAction {
  Ec2Stop = "ec2-stop",
  Ec2Restart = "ec2-restart",
  DetachVolume = "detach-volume",
}
```

But unfortunately at last we only have:

```ts
export interface V1alpha1AWSChaosSpec {
  /**
   * @type {string}
   * @memberof V1alpha1AWSChaosSpec
   */
  action?: string;

  //...
}
```

This prevents us from using the TS compiler API to read the enum, but the solution
is also simple, **since we can't get the key information through the code, we need
to define it manually**.

How about defining a json file? This is what I first thought, but @STRRL reminded
me that **we can write it in the comments**. Yes, like `kubebuilder`'s marker, we
can also define markers.

So the following are all markers we will using:

- ui:form:enum=action1;action2;action3

  > Indicate what actions we have.

- ui:form:action=action1

  > Indicate this property belongs to which action.

- ui:form:ignore

  > Ignore this property.

For example:

```go
type AWSChaosSpec struct {
        // ui:form:enum:ec2-stop,ec2-restart,detach-volume
        // +kubebuilder:validation:Enum=ec2-stop;ec2-restart;detach-volume
        Action AWSChaosAction `json:"action"`

        //...
}

type AWSSelector struct {
        // Endpoint indicates the endpoint of the aws server. Just used it in test now.
        // ui:form:ignore
        // +optional
        Endpoint *string `json:"endpoint,omitempty"`

        //...

        // DeviceName indicates the name of the device.
        // ui:form:action:detach-volume
        // +optional
        DeviceName *string `json:"deviceName,omitempty" webhook:"AWSDeviceName,nilable"`
}
```

The rest of the steps are simple, use regular expressions to read them:

```js
// Part of the code
const UI_FORM_ENUM = /ui:form:enum:(.+)\s/;

/**
 * Get enum array from jsdoc comment.
 *
 * @export
 * @param {string} s
 * @return {string[]}
 */
export function getUIFormEnum(s) {
  const matched = s.match(UI_FORM_ENUM);

  return matched ? matched[1].split(",") : [];
}

// ...

// Assuming that the Node is found
const { escapedText: identifier } = node.name; // identifier
const comment = node.jsDoc[0].comment;

if (identifier === "action") {
  // Get all actions
  actions = getUIFormEnum(comment);
}
```

Similarly we can handle other markers in this way. So far we have solved this problem.

#### Shared fields

Now this problem also becomes simple, since we have defined markers, we can default
to:

**The fields without `ui:form:action:xxx` and `ui:form:ignore` are shared**.

I think this part can be ignored for code details, it is enough to understand this
given condition.

#### Chaos without action

Here is another case, what if a chaos doesn't have an action? Like `KernelChaos`
and `TimeChaos`.

This Chaos will output `export default shared` directly.

#### Non-primitive type

If a field is not primitive type, TypeScript will use an `TypeReference` to represent
it. This is different from primitive types, once the type reference appears, we
need to recursively get the corresponding type symbol. We will use the checker to
achieve this:

```ts
const program = ts.createProgram([source], {
  target: ts.ScriptTarget.ES2015,
});
const sourceFile = program.getSourceFile(source);
const checker = program.getTypeChecker(); // this is what we need

const type = checker.getTypeAtLocation(typeRef); // get the final type
```

So we still need to use a new field type to represent it. Here is an example:

```js
{
    field: "ref",
    label: "callchain",
    multiple: true,
    children: [
        {
            field: "text",
            label: "funcname",
            value: "",
            helperText: ""
        },
        {
            field: "text",
            label: "parameters",
            value: "",
            helperText: ""
        },
        {
            field: "text",
            label: "predicate",
            value: "",
            helperText: ""
        }
    ]
},
```

When a field is represented as a `ref`, then it must contain the `children` key,
which means it will render a children group to the interface.

There is also a `mutiple` key to indicate whether children are rendered repeatedly.

#### Result

Finally, we will generate `AWSChaos` like this:

```ts
/**
 * This file was auto-generated by @ui/openapi.
 * Do not make direct changes to the file.
 */

const shared = [
  {
    field: "text",
    label: "awsRegion",
    value: "",
    helperText: "AWSRegion defines the region of aws.",
  },
  {
    field: "text",
    label: "ec2Instance",
    value: "",
    helperText: "Ec2Instance indicates the ID of the ec2 instance.",
  },
  {
    field: "text",
    label: "secretName",
    value: "",
    helperText: "Optional. SecretName defines the name of kubernetes secret.",
  },
];

export default {
  "ec2-stop": shared,
  "ec2-restart": shared,
  "detach-volume": [
    ...shared,
    {
      field: "text",
      label: "deviceName",
      value: "",
      helperText:
        "Optional. DeviceName indicates the name of the device. Needed in detach-volume.",
    },
    {
      field: "text",
      label: "volumeID",
      value: "",
      helperText:
        "Optional. EbsVolume indicates the ID of the EBS volume. Needed in detach-volume.",
    },
  ],
};
```

Also `KernelChaos`:

```ts
const shared = [
  {
    field: "ref",
    label: "failKernRequest",
    children: [
      {
        field: "ref",
        label: "callchain",
        multiple: true,
        children: [
          {
            field: "text",
            label: "funcname",
            value: "",
            helperText: "xxx",
          },
          {
            field: "text",
            label: "parameters",
            value: "",
            helperText: "xxx",
          },
          {
            field: "text",
            label: "predicate",
            value: "",
            helperText: "xxx",
          },
        ],
      },
      {
        field: "text",
        label: "failtype",
        value: 0,
        helperText: "xxx",
      },
      {
        field: "label",
        label: "headers",
        value: [],
        helperText: "xxx",
      },
      {
        field: "text",
        label: "probability",
        value: 0,
        helperText: "xxx",
      },
      {
        field: "text",
        label: "times",
        value: 0,
        helperText: "xxx",
      },
    ],
  },
];

export default shared;
```

But this is not an ideal result and there are still some details that need to be
addressed, like:

- ~~Some useless markers remain in the `helperText`.~~
- ~~If an action only uses shared fields, the spread operator will be excess.~~
- Output components directly? (enhancement)

These can be followed up as optimization matters.

## Drawbacks

The biggest disadvantage of automation is that we generate a lot of useless structures,
but luckily we can fix this in the compilation phase.

There is also the fact that we have to adapt the existing code to what the automation
generates, which can still be a lot of work. Automation solves the code burden,
but it can become difficult to debug.

## Alternatives

There still have a way to do these things, use Go code to generate the code, but
it is not the best way because: **It's only benefit the part of converting native
types to typescript types, but it's hard to generate real typescript
types (Write from scratch)**.

## Unresolved questions

Already described in [Result](#result).
