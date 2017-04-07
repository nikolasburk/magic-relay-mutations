# The Magic Behind Relay Mutations

[Relay](https://facebook.github.io/relay/) is a powerful GraphQL client for React and React Native applications. It was open sourced by Facebook alongside GraphQL in 2015 and basically takes care of anything that concerns an app's data layer.

In this post, we are going to explore how Relay mutations work by the example of a React Native app - check out the code on [GitHub](https://github.com/graphcool-examples/react-native-relay-pokedex-example) if you like. The sample application is a simple Pokedex app, where users can manage their Pokemons.

![](http://i.imgur.com/S21GfEo.png)

> Note: We're going to assume a basic familiarity with GraphQL in this article. If you haven't heard of GraphQL before, the [documentation](www.graphql.org) or the [GraphQL for iOS Developers](http://artsy.github.io/blog/2016/06/19/graphql-for-mobile/) post are great places to start. If you're interested in learning more about Relay in general, head over to [Learn Relay](www.learnrelay.org) for a comprehensive tutorial.

If you want to run the example with your own GraphQL server, you can use [graphql-up](https://www.graph.cool/graphql-up/) to quickly spin one up yourself.

[![graphql-up](http://static.graph.cool/images/graphql-up.svg)](https://www.graph.cool/graphql-up/new?source=https://raw.githubusercontent.com/graphcool-examples/react-native-relay-pokedex-example/master/pokedex.schema)

## Recap: Mutations

In GraphQL, a _mutation_ is the only way to create, update or delete data on the server - they effectively are the GraphQL abstraction for performing _writes_ in the database. 

As an example, creating a new Pokemon in our sample app uses the following mutation:

```graphql
mutation CreatePokemonMutation($name: String!, $url: String!) {
  createPokemon(name: $name, url: $url) {
    id # will be returned by the server
  }
}
```

Notice that mutations, similar to queries, also require a _payload_ to be specified, representing the information that we'd like to have returned from the server after the mutation was performed. In the above example, we're asking for the `id` of the new Pokemon.


## Relay - A brief Overview

Relay is the most sophisticated GraphQL client available at the moment. Like GraphQL, it has been used internally by Facebook for a while before being open sourced.

Relay surely isn't the easiest framework to learn (even the Relay team says so), but when used correctly, it manages the whole data layer for you in a consistent and reliable manner. It therefore is particularly well suited for large-scale and complex applications and ensures long-term developer productivity in these contexts.

### Declarative API and Colocation

With Relay, React components specify their data requirements in a declarative fashion, making use of [GraphQL fragments](http://graphql.org/learn/queries/#fragments).

Considering the `PokemonDetails` view above, we need to display the Pokemon's name and image, the fragment that represents our data requirements thus looks as follows:

```graphql
fragment PokemonDetails on Node {
  ... on Pokemon {
    id
    name
    url
  }
}
```
Note that the `id` is required to perform an update or deletion, so it's requested as well.

These fragments are usually kept in the same file as the React component, so UI and data requirements are _colocated_. Relay then uses a [higher-order component](https://facebook.github.io/react/docs/higher-order-components.html) called [`Relay.Container`](https://facebook.github.io/relay/docs/guides-containers.html#content), to wrap the component along with its data requirements - from this point the developer doesn't have to worry about the data any more! It will be fetched behind the scenes and is made available to the component through its props.


### Data Masking

Another core concept of Relay is [data masking](https://facebook.github.io/relay/docs/thinking-in-relay.html#data-masking), which means that any component will only ever have access to the data that it explicitly requests in a fragment. The data requirements are passed upwards through the component tree, where at the top they're combined a in a _root query_. Relay also makes sure that only data that has not been requested before is being fetched, so there is a lot of optimizations happening to ensure excellent performance and minimal data transfer over the network.

Consider the example of the `PokemonList` and the `PokemonItem` (representing a single item, or _cell_ in the list):

```graphql
Relay.createContainer(PokemonList, {
  fragments: {
    viewer: () => Relay.QL`
      fragment on Viewer {
        id
        allPokemons(first: 10) {
          edges {
            node {
              ${PokemonItem.getFragment('pokemon')}
            }
          }
        }
      }
    `
  }
})

Relay.createContainer(PokemonItem, {
  fragments: {
    pokemon: () => Relay.QL`
      fragment PokemonDetails on Pokemon {
        id
        name
        url
      }
    `
  }
})
```

Despite the fact that the `PokemonList` indirectly requests the Pokemons' `id`, `name` and `url` (since it incorporates the fragment of the `PokemonItem`), it won't be able to access this information in its props!


## Relay Mutations

Relay doesn't give the developer the ability to manually modify the data that it stores internally. Instead, with every change, it requires a _description_ of how the local cache should be updated after the change happened in the form of a [mutation](https://facebook.github.io/relay/docs/guides-mutations.html#content) and then takes care of the update itself.

The description is provided by subclassing `Relay.Mutation` and implementing (at least) four abstract methods that help Relay to properly update the local store:

- `getMutation()`: the name of the mutation (from the GraphQL schema)
- `getVariables()`: the input variables for the mutation
- `getFatQuery()`: a GraphQL query that fetches all data that potentially was changed due to the mutation
- `getConfigs()`: a precise specification how the mutation should be incorporated into the cache

In the following, we'll take a deeper look at the mutations in our sample app, which are for creating, updating and deleting Pokemons.

> Note: We're using the [Graphcool Relay API](https://www.graph.cool/docs/reference/relay-api/overview-aizoong9ah) for this example. If you used `graphql-up` to create your own backend, you can explore the API by pasting the endpoint for the Relay API into the address bar of a browser.

### Creating a new Pokemon

![](http://i.imgur.com/yskx5KU.png)

Let's walk through the different methods and understand what information we have to provide so that Relay can successfully merge the newly created Pokemon into its store.

The first two methods, `getMutation()` and `getVariables()` are relatively obvious and can be retrieved directly from the documentation where the API is described.

The implementations look as follows:

```
  getMutation() {
    return Relay.QL`mutation {createPokemon}`
  }

  getVariables() {
    const {pokemonName, pokemonUrl} = this.props 
    return {
      name: pokemonName,
      url: pokemonUrl
    }
  }
```

Notice that the `props` of a `Relay.Mutation` are passed to it through its constructor. Here, we simply provide the `name` and the `url` of the Pokemon that is to be created.

Now, on to the interesting parts. In `getFatQuery()`, we need to specify the parts that might change due to the mutation. Here, we simply specify the `viewer`:

```
getFatQuery() {
  return Relay.QL`
    fragment on CreatePokemonPayload {
      viewer {
        allPokemons
      }
    }
  `
}
```

Notice that _all_ subfields of `allPokemons` are also automatically included with this approach. `allPokemons` really is the only point we expect to change after our mutation is performed.

Finally, in `getConfigs()`, we need to specify the [mutator configurations](https://facebook.github.io/relay/docs/guides-mutations.html#mutator-configuration), telling Relay exactly how the new data should be incorporated into the cache:

```
getConfigs() {
  return [{
    type: 'RANGE_ADD',
    parentName: 'viewer',
    parentID: this.props.viewerId,
    connectionName: 'allPokemons',
    edgeName: 'edge',
    rangeBehaviors: {
      '': 'append'
    }
  }]
}
```

We first express that we want to _add_ the node using `RANGE_ADD` for the `type` (there are 5 different types in total). 

Relay internally represents the stored data as a graph, so the remaining information expresses where exactly the new node should be hooked into the existing structure.

Let's consider the shape of the data before we move on:

```
viewer {
  allPokemons {
    edges {
      node {
        id
        name
      }
    }
  }
}
```

Here we clearly see the direct connection between `viewer` and the Pokemons goes through `allPokemons` connection, so the _parent_ of the new Pokemon is the `viewer`. The name of that connection is `allPokemons`, and lastly the `edgeName`  is taken from the payload of the mutation.

The last piece, `rangeBehaviors`, specifies whether we want to _append_ or _prepend_ the new node.

Sending the mutation is as simple as calling `commitUpdate` on the `relay` prop that is injected to each component that is wrapped with a `Relay.Container`:

```
_sendCreatePokemonMutation = () => {
  const createPokemonMutation = new CreatePokemonMutation({
    viewerId: this.props.viewer.id,
    pokemonName: this.state.pokemonName,
    pokemonUrl: this.state.pokemonUrl,
  })
  this.props.relay.commitUpdate(createPokemonMutation)
}
```  

### Updating a Pokemon

Like with creating a Pokemon, `getMutation()` and `getVariables()` are trivial to implement and can be derived directly from the documentation:

```
getMutation() {
  return Relay.QL`mutation {updatePokemon}`
}

getVariables() {
  const {pokemonId, pokemonName, pokemonUrl} = this.props
  return {
    id: pokemonId,
    name: pokemonName,
    url: pokemonUrl
  }
}
```

In `getFatQuery()`, we only include the `pokemon` which includes the updated info this time, since that is the only part we expect to change after our mutation:

```
getFatQuery() {
  return Relay.QL`
    fragment on UpdatePokemonPayload {
      pokemon
    }
  `
}
```

Finally, `getConfigs()`, this time specifies a mutator configuration of type `FIELDS_CHANGE` since we're only updating properties on a single Pokemon:

```
getConfigs() {
  return [{
    type: 'FIELDS_CHANGE',
    fieldIDs: {
      pokemon: this.props.pokemonId,
    }
  }]
}
```

As only additional piece of info, we declare the ID of the Pokemon that is being updated so that Relay has this information available when receiving the new Pokemon data.


### Deleting a Pokemon

As before, `getMutation()` and `getVariables()` are self-explanatory:

```
getMutation() {
  return Relay.QL`mutation { deletePokemon }`
}

getVariables() {
  return {
    id: this.props.pokemonId,
  }
}
```

Then, in `getFatQuery()`, we need to retrieve the `pokemon` from the mutation payload:

```
getFatQuery() {
  return Relay.QL`
    fragment on DeletePokemonPayload {
      pokemon
    }
  `
}
```

In `getConfigs()`, we're getting to know another mutator configuration type called `NODE_DELETE`. This one requires a `parentName` as well as a `connectionName`, both coming from the mutation payload and specifying where that node existed in Relay's data graph. Another requirement, that is specifically relevant for the implementation of a GraphQL server, is that the mutation payload of a deleting mutation always needs to return the `id` of the node that was deleted so that Relay can find that node in its store. Taking all of this together, our implementation of `getConfigs()` can be written like so:

```
getConfigs() {
  return [{
    type: 'NODE_DELETE',
    parentName: 'pokemon',
    connectionName: 'edge',
    deletedIDFieldName: 'deletedId'
  }]
}
```
