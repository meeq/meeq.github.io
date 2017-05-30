---
layout: post
title: Bi-Directional Cursor Pagination with React.js, Relay, and GraphQL
category: Software
tags:
  - React
  - GraphQL
  - Relay
  - Programming
  - JavaScript
---
###### [This post was also published on HackerNoon, an online publication for software engineers.](https://hackernoon.com/bi-directional-cursor-pagination-with-react-js-relay-and-graphql-dc4ad6f9cbb0)

Pagination comes in many different flavors depending on the desired user experience and the shape of the underlying API. GraphQL APIs such as [GitHub’s](https://developer.github.com/early-access/graphql/) implement the [Relay Cursor Connections Specification](https://facebook.github.io/relay/graphql/connections.htm) to standardize pagination and slicing of large result sets. This approach is well-suited to infinite-scrolling, but can also be used for “windowed” paging with next/previous page buttons.

![An example of bi-directional windowed paging using the GitHub GraphQL API]({{ site.url }}/assets/bidirectional-pagination-with-relay.png)

Cursor connections work by passing in one of the following query argument pairs:

### Forward Windowed Pagination

`first` is a positive, non-zero integer describing the maximum number of results to return from the leading side of the results set. During backward-pagination, this value must be null.

`after` is an opaque cursor type value provided by the `endCursor` field of the connection’s [`pageInfo`](https://developer.github.com/early-access/graphql/object/pageinfo/) object. For the first page, this value will be null. During backward-pagination, this value must be null.

### Backward Windowed Pagination

`last` is a positive, non-zero integer describing the maximum number of results to return from the trailing side of the results set. During forward-pagination, this value must be null.

`before` is an opaque cursor type value provided by the `startCursor` field of the connection’s [`pageInfo`](https://developer.github.com/early-access/graphql/object/pageinfo/) object. During forward-pagination, this value must be null.

### GraphQL Query fragment example

    Relay.QL`
      fragment on Query {
        search(query: $q, type: REPOSITORY,
               first: $first, after: $after,
               last: $last, before: $before) {
          repositoryCount
          pageInfo {
            startCursor
            endCursor
          }
          edges {
            node {
              ... on Repository {
                id
                name
                url
              }
            }
          }
        }
      }
    `

The `repositoryCount` field is used to calculate the total number of result pages. [`pageInfo`](https://developer.github.com/early-access/graphql/object/pageinfo/) is used to populate the `before`/`after` cursor arguments. [`edges`](https://developer.github.com/early-access/graphql/object/searchresultitemedge) contains the search results inside the [`node`](https://developer.github.com/early-access/graphql/union/searchresultitem/) Union type. We have restricted the results to only contain [`repository`](https://developer.github.com/early-access/graphql/object/repository/) type nodes using the `type` argument.

Using the data from this query, the application can calculate the total number of pages and keep track of which page we are currently on. The page count is the rounded-up result of dividing the total number of results by the number of results per page.

To see the next page, run the query using the `endCursor` field of the current page as the `after` argument and the page size as the `first` argument. To go back a page, run the query using the `startCursor` field of the current page as the `before` argument and the page size as the `last` argument.

### Encapsulating pagination logic in a React Component

    class Search extends Component {

In order to propagate the query results through a [Relay Container](https://facebook.github.io/relay/docs/guides-containers.html), we need to create a [React Component](https://facebook.github.io/react/docs/react-component.html) to process, act on, and display the data.

      constructor(props) {
        super(props);

        this.state = {
          q: "",
          pageSize: 25,
          pageCursor: 1,
          pageCount: 1
        };

        this.handleQueryChange = this.handleQueryChange.bind(this);
        this.handleSearch = this.handleSearch.bind(this);
        this.handlePrevPage = this.handlePrevPage.bind(this);
        this.handleNextPage = this.handleNextPage.bind(this);
      }

The `constructor` is mostly boilerplate with some initial state and function bindings to ensure the component is the function context during callbacks.

      handleQueryChange(e) {
        this.setState({
          q: e.target.value
        });
      }

`handleQueryChange` is called by an &lt;input&gt; `onChange` callback to update the search query based on user input. The actual search does not happen until the form is submitted.

      handleSearch(e) {
        this.props.relay.setVariables({
          q: this.state.q,
          first: this.state.pageSize,
          after: null,
          last: null,
          before: null
        });
        this.isNewSearchQuery = true;
        e && e.preventDefault(); // Stop form action
      }

`handleSearch` is called by the &lt;form&gt; `onSubmit` callback to perform the search by updating the variables in the Relay query. The `isNewSearchQuery` flag is used by `componentWillReceiveProps` as a cue to reset the page number and page count when the query value changes.

      componentWillReceiveProps(nextProps) {
        if (this.isNewSearchQuery) {
          let repoCount = nextProps.query.search.repositoryCount;
          let reposPerPage = this.state.pageSize
          this.setState({
            pageCursor: 1,
            pageCount: Math.ceil(repoCount / reposPerPage)
          });
          this.isNewSearchQuery = false;
        }
      }

[`componentWillReceiveProps`](https://facebook.github.io/react/docs/react-component.html#componentwillreceiveprops) is a React Component hook that is called when the query results are returned. From here, we can check if the `isNewSearchQuery` flag was set and reset the page cursor to the beginning and re-calculate the page count. This pattern works best when the total number of results does not wildly change during pagination.

      handlePrevPage() {
        if (this.state.pageCursor > 1) {
          let pageCursor = this.state.pageCursor - 1;
          this.props.relay.setVariables({
            last: this.state.pageSize,
            before: this.props.query.search.pageInfo.startCursor,
            first: null,
            after: null
          }, ({ ready, done }) => {
            if (ready && done) {
              this.setState({ pageCursor });
            }
          });
        }
      }

      handleNextPage() {
        if (this.state.pageCursor < this.state.pageCount) {
          let pageCursor = this.state.pageCursor + 1;
          this.props.relay.setVariables({
            first: this.state.pageSize,
            after: this.props.query.search.pageInfo.endCursor,
            last: null,
            before: null
          }, ({ ready, done }) => {
            if (ready && done) {
              this.setState({ pageCursor });
            }
          });
        }
      }

`handlePrevPage` and `handleNextPage` perform bounds checking based on the pagination state, then update the Relay query variables using the current results’ [`pageInfo`](https://developer.github.com/early-access/graphql/object/pageinfo/). The actual page number does not update until the new query has completed by utilizing the optional `onReadyStateChange` argument of [`setVariables`](https://facebook.github.io/relay/docs/api-reference-relay-container.html#setvariables).

      render() {
        let { search } = this.props.query;
        let { pageCursor, pageCount } = this.state;
        let isPrevPageDisabled = (pageCursor === 1);
        let isNextPageDisabled = (pageCursor === pageCount);
        return (
          <div>
            <h2>Search</h2>

            <form onSubmit={this.handleSearch}>
              <input placeholder="Repository name" type="text"
                     value={this.state.q} onChange={this.handleQueryChange} />
              &nbsp;
              <button type="submit">Search</button>
            </form>

            <h3>{search.repositoryCount} results</h3>

            <button onClick={this.handlePrevPage}   
                    disabled={isPrevPageDisabled}>Previous Page</button>
            &nbsp;
            Page {pageCursor} of {pageCount}
            &nbsp;
            <button onClick={this.handleNextPage}
                    disabled={isNextPageDisabled}>Next Page</button>

            <ul className="Search-results">
              {search.edges.map(({ node }, index) => (
                <SearchResult key={node.id} repository={node} />
              ))}
            </ul>
          </div>
        );
      }
    }

`render` creates the DOM for the Search component, hooking up all of the callbacks and displaying the results by mapping through the [`edges`](https://developer.github.com/early-access/graphql/object/searchresultitemedge) and encapsulating the search result items in their own component:

    class SearchResult extends Component {
      render() {
        let repo = this.props.repository;
        let stargazersUrl = repo.url + "/stargazers";
        return (
          <li>
            <h5 className="stargazers"><a href={stargazersUrl}>★ {repo.stargazers.totalCount}</a></h5>
            <h4 className="headline"><a href={repo.owner.url}>{repo.owner.login}</a>/<a href={repo.url}>{repo.name}</a></h4>
            <span className="description">{repo.description}</span>
          </li>
        )
      }
    }

### Wrapping it all up with a Relay Container

Finally, we can create a Relay Container based on our query fragments and React Component:

    Relay.createContainer(Search, {
      initialVariables: {
        q: "",
        first: null,
        last: null,
        before: null,
        after: null
      },
      fragments: {
        query: () => Relay.QL`
          fragment on Query {
            search(query: $q, type: REPOSITORY,
                   first: $first, after: $after,
                   last: $last, before: $before) {
              repositoryCount
              pageInfo {
                startCursor
                endCursor
              }
              edges {
                node {
                  ... on Repository {
                    id
                    name
                    url
                    description
                    stargazers {
                      totalCount
                    }
                    owner {
                      login
                      url
                    }
                  }
                }
              }
            }
          }
        `,
      },
    });

This container can be passed into a [Relay Renderer](https://facebook.github.io/relay/docs/api-reference-relay-renderer.html) or used as a child-container inside of a parent with the [`getFragment`](https://facebook.github.io/relay/docs/api-reference-relay-container.html#getfragment) static method.

I hope this was helpful for beginners trying to wrap their head around the GraphQL pagination model and integrate with React and Relay Classic. I wanted to share this because I could not find any examples of bi-directional pagination in GraphQL while solving this exercise.

Thanks for reading.
