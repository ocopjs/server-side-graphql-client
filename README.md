<!--[meta]
section: api
title: server-side graphql client
[meta]-->

# server-side graphql client

OcopJS - Để sử dụng truy vấn GraphQL API từ chính máy chủ.

> Lưu ý sau khi phiên bản KeystoneJS 5 chuyển sang chế độ duy trì để ra mắt
> phiên bản mới hơn. Chúng tôi đã dựa trên mã nguồn cũ này để phát triển một
> phiên bản khác với một số tính năng theo hướng microservices.

Thư viện này xử dụng hàm `ocop.executeGraphQL` để phát triển thêm, giúp việc
truy cấn ở máy chủ dễ hơn. Không phải viết câu truy vấn GraphQL và gọi thông
thường.

Tạo người dùng mới thông thường sẽ được thực hiện bằng `ocop.executeGraphQL` như
sau:

```js
const { data, errors } = await ocop.executeGraphQL({
  query: `mutation ($item: createUserInput){
    createUser(data: $item) {
      id
      name
    }
  }`,
  variables: { item: { name: "alice" } },
});
const user = data.createUser;
```

Xài thư viện `@ocopjs/server-side-graphql-client` này chúng ta có thể viết gọn hơn
là:

```js
const { createItem } = require("@ocopjs/server-side-graphql-client");

const user = await createItem({
  ocop,
  listkey: "User",
  item: { name: "alice" },
  returnfields: `id name`,
});
```

There are three key differences between `ocop.executeGraphQL` and `createItem`
(and other functions from this package):

1. If there is an error, `createItem` will be thrown as an exception, rather
   than providing the error as a return value.
2. `createItem` runs with _access control disabled_. This is suitable for use
   cases such as seeding data or other server side scripts where the query is
   triggered by the system, rather than a specific user. This can be controlled
   with the `context` option.
3. All queries are internally paginated and all mutations are internally
   chunked. This can be controlled with the `pageSize` option.

### Use cases

These utilities can be used for a wide range of specific use-cases, some more
common examples might include simple data seeding:

```js
const seedUsers = async (usersData) => {
  await createItems({ ocop, listKey: "User", items: usersData });
};
```

or fetching data inside hooks:

```js
// This example will copy data from a related field if set
ocop.createList("Page", {
  fields: {
    name: { type: Text },
    content: { type: Text },
    copy: { type: Relationship, ref: "Page" },
  },
  hooks: {
    resolveInput: async ({ resolvedData }) => {
      // Whenever copy field is set fetch the related data
      const pageToCopy = resolvedData.copy
        ? await getItem({
            ocop,
            listKey: "Page",
            itemId: resolvedData.copy,
            returnFields: "name, content",
          })
        : {};
      // resolve data from the copied item and unset the relationship
      return { ...resolvedData, ...pageToCopy, copy: undefined };
    },
  },
});
```

## API

To perform CRUD operations, use the following functions:

- [`createItem`](#createitem)
- [`createItems`](#createitems)
- [`getItem`](#getitem)
- [`getItems`](#getitems)
- [`updateItem`](#updateitem)
- [`updateItems`](#updateitems)
- [`deleteItem`](#deleteitem)
- [`deleteItems`](#deleteitems)

For custom queries use [`runCustomQuery`](#runcustomquery).

> NOTE: All functions accept a config object as an argument, and return a
> `Promise`.

### Shared Config Options

The following config options are common to all server-side graphQL functions.

| Properties     | Type     | Default    | Description                                                                                                                                                                                                         |
| -------------- | -------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ocop`         | `Ocop`   | (required) | Ocop instance.                                                                                                                                                                                                      |
| `listKey`      | `String` | (required) | Ocop list name.                                                                                                                                                                                                     |
| `returnFields` | `String` | `id`       | A graphQL fragment of fields to return. Must match the graphQL return type.                                                                                                                                         |
| `context`      | `Object` | N/A        | An Apollo [`context` object](https://www.apollographql.com/docs/apollo-server/data/resolvers/#the-context-argument). See the [server side graphQL docs](/docs/discussions/server-side-graphql.md) for more details. |

> NOTE: If `context` argument is provided then the `ocop` argument is not
> required.

### `createItem`

Create a single item.

#### Usage

```js
const { createItem } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const addUser = async (userInput) => {
  const user = await createItem({
    ocop,
    listKey: "User",
    item: userInput,
    returnFields: `name, email`,
  });
  console.log(user); // { name: 'ocop user', email: 'ocop@test.com'}
};

addUser({ name: "ocop user", email: "ocop@test.com" });
```

**Note**: The `item` property is a graphQL create input. For Relationship fields
it can contain nested mutations with create and connect operations. For examples
see the
[Relationship API documentation](/packages/fields/src/types/Relationship/README.md#create-and-append-a-related-item).

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type                           | Default    | Description             |
| ---------- | ------------------------------ | ---------- | ----------------------- |
| `item`     | GraphQL `[listKey]CreateInput` | (required) | The item to be created. |

### `createItems`

Create multiple items.

#### Usage

```js
const { createItems } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const dummyUsers = [
  { data: { name: "user1", email: "user1@test.com" } },
  { data: { name: "user2", email: "user2@test.com" } },
];

const addUsers = async () => {
  const users = await createItems({
    ocop,
    listKey: "User",
    items: dummyUsers,
    returnFields: `name`,
  });
  console.log(users); // [{name: `user2`}, {name: `user2`}]
};
addUsers();
```

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type                            | Default    | Description                                                                                    |
| ---------- | ------------------------------- | ---------- | ---------------------------------------------------------------------------------------------- |
| `items`    | GraphQL `[listKey]sCreateInput` | (required) | The array of objects to be created.                                                            |
| `pageSize` | `Number`                        | 500        | The create mutation batch size. This is useful when you have large set of data to be inserted. |

### `getItem`

Retrieve a single item by its ID.

#### Usage

```js
const { getItem } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const getUser = async ({ itemId }) => {
  const user = await getItem({
    ocop,
    listKey: "User",
    itemId,
    returnFields: "id, name",
  });
  console.log(user); // User 123: { id: '123', name: 'Aman' }
};
getUser({ itemId: "123" });
```

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type     | Default    | Description                           |
| ---------- | -------- | ---------- | ------------------------------------- |
| `itemId`   | `String` | (required) | The `id` of the item to be retrieved. |

### `getItems`

Retrieve multiple items. Use
[where](https://www.ocop.vn/guides/intro-to-graphql/#where) clause to filter
results.

#### Usage

```js
const { getItems } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const getUsers = async () => {
  const allUsers = await getItems({
    ocop,
    listKey: "User",
    returnFields: "name",
  });
  const someUsers = await getItems({
    ocop,
    listKey: "User",
    returnFields: "name",
    where: { name: "user1" },
  });
  console.log(allUsers); // [{ name: 'user1' }, { name: 'user2' }];
  console.log(someUsers); // [{ name: 'user1' }];
};
getUsers();
```

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type                             | Default | Description                                                                                                                                 |
| ---------- | -------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `where`    | GraphQL `[listKey]WhereInput`    | `{}`    | Limit results to items matching [where clause](https://www.ocop.vn/guides/intro-to-graphql/#where).                                         |
| `sortBy`   | GraphQL enum `Sort[listKey]sBy}` |         | Returned the results based on specified order. Refer [docs](https://www.ocop.vn/guides/intro-to-graphql#sortby) for available sort options. |
| `first`    | `Number`                         |         | Limit the number of items returned from the query.                                                                                          |
| `skip`     | `Number`                         |         | Skip that many elements in the list before collecting the items to be returned.                                                             |
| `pageSize` | `Number`                         | 500     | The query batch size. Useful when retrieving a large set of data.                                                                           |

### `updateItem`

Update a single item.

#### Usage

```js
const { updateItem } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const updateUser = async (updateUser) => {
  const updatedUser = await updateItem({
    ocop,
    listKey: "User",
    item: updateUser,
    returnFields: "name",
  });
  console.log(updatedUser); // { name: 'newName'}
};
updateUser({ id: "123", data: { name: "newName" } });
```

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type                           | Default    | Description             |
| ---------- | ------------------------------ | ---------- | ----------------------- |
| `item`     | GraphQL `[listKey]UpdateInput` | (required) | The item to be updated. |

### `updateItems`

Update multiple items.

#### Usage

```js
const { updateItems } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const updateUsers = async (updateUsers) => {
  const users = await updateItems({
    ocop,
    listKey: "User",
    items: updateUsers,
    returnFields: "name",
  });

  console.log(users); // [{name: 'newName1'}, {name: 'newName2'}]
};

updateUsers([
  { id: "123", data: { name: "newName1" } },
  { id: "456", data: { name: "newName2" } },
]);
```

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type                            | Default    | Description                                                               |
| ---------- | ------------------------------- | ---------- | ------------------------------------------------------------------------- |
| `items`    | GraphQL `[listKey]sUpdateInput` | (required) | Array of items to be updated.                                             |
| `pageSize` | `Number`                        | 500        | The update mutation batch size. Useful when updating a large set of data. |

### `deleteItem`

Delete a single item.

#### Usage

```js
const { deleteItem } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const deleteUser = async (itemId) => {
  const user = await deleteItem({ ocop, listKey: "User", itemId });
  console.log(user); // { id: '123' }
};
deleteUser("123");
```

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type     | Default    | Description                         |
| ---------- | -------- | ---------- | ----------------------------------- |
| `itemId`   | `String` | (required) | The `id` of the item to be deleted. |

### `deleteItems`

Delete multiple items.

#### Usage

```js
const { deleteItems } = require("@ocop/server-side-graphql-client");

ocop.createList("User", {
  fields: {
    name: { type: Text },
    email: { type: Text },
  },
});

const deletedUsers = async (items) => {
  const users = await deleteItems({ ocop, listKey: "User", items });
  console.log(users); // [{id: '123'}, {id: '456'}]
};
deletedUsers(["123", "456"]);
```

#### Config

[Shared Config Options](#shared-config-options) apply to this function.

| Properties | Type       | Default    | Description                                                               |
| ---------- | ---------- | ---------- | ------------------------------------------------------------------------- |
| `items`    | `String[]` | (required) | Array of item `id`s to be deleted.                                        |
| `pageSize` | `Number`   | 500        | The delete mutation batch size. Useful when deleting a large set of data. |

### `runCustomQuery`

Execute a custom query.

#### Config

| Properties  | Type     | Default    | Description                                                                                                                                                                                                         |
| ----------- | -------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ocop`      | Object   | (required) | Ocop instance.                                                                                                                                                                                                      |
| `query`     | String   | (required) | The GraphQL query to execute.                                                                                                                                                                                       |
| `variables` | Object   | (required) | Object containing variables your custom query needs.                                                                                                                                                                |
| `context`   | `Object` | N/A        | An Apollo [`context` object](https://www.apollographql.com/docs/apollo-server/data/resolvers/#the-context-argument). See the [server side graphQL docs](/docs/discussions/server-side-graphql.md) for more details. |
