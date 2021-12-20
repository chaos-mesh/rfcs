# Scalable codebase in UI via monorepo

## Summary

Use [Yarn Workspaces](https://classic.yarnpkg.com/lang/en/docs/workspaces/)
to easily extend the front-end codebase.

## Motivation

> Why are we doing this?

As the project grows, so does the number of features that need to be supported
on the front-end. But the current architecture of front-end is a single-repo, which
means all dependencies are intertwined. With this system, isolating applications
and dependencies is difficult. You can only put the common logic under a certain
folder, like `lib` or `components`, it's not possible to put the logic out of the
`src` folder.

So we decided to use monorepo to separate the front-end codebase.

> What use cases does it support?

According to Chaos Mesh itself, in the future, the front-end will plan to generate
the corresponding typescript definitions through CRDs. Code like this is application
agnostic, and putting them together with the original code can be a huge hindrance
to testing and management.

> What is the expected outcome?

Before monorepo, the codebase looks like this:

```shell
ui/
├── src/
│   ├── api/
│   ├── components/
│   ├── components-mui/
│   ├── lib/
│   ├── pages/
├── package.json
```

After using monorepo:

```shell
ui/
├── app/ # original src
│   ├── api/
│   ├── components/
│   ├── lib/
│   ├── pages/
├── packages/
│   ├── mui-extends/ # components-mui
│   ├── crd/ # generate ts code from CRDs
├── package.json
```

## Detailed design

First, it's needed to treat the current `src` folder as a package, we decided to
rename it to `app` and move it to `ui/app`.

Then extract the public logic, like `husky` and `lint-staged`, we will use them
in the root:

```shell
yarn add husky lint-staged prettier prettier-plugin-import-sort import-sort-style-eslint -DW # --dev --ignore-workspace-root-check
```

Then add the below into `ui/package.json`:

```json
{
   "workspaces": [
    "app",
    "packages/*"
  ]
}
```

Finally, we are ready to create packages:

```shell
mkdir -p packages/mui-extends
cd packages/mui-extends
npm init
```

> Note:
>
> Some commands will be changed, like:
>
> - `yarn start:default` -> `yarn workspace @ui/app start:default`

## Drawbacks

A good explanation:

https://fossa.com/blog/pros-cons-using-monorepos/

## Alternatives

There should be no better way to manage future non-application related logic, after
all, the front-end code is also attached to the entire chaos-mesh codebase.

## Unresolved questions

No.
