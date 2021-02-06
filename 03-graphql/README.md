# Exercise 2 - GraphQL

GraphQL is a query language that offers an alternative model to developing APIs (REST, SOAP or gRPC) with detailed description.

We are going to build a Spring GraphQL example to apply different operations on articles. 

## 🔧 Step 1 - Setup GraphQL Server

### Dependencies

We include the following dependencies in order to be able to start writing the GraphQL schema.

```xml
<dependencies>

    <dependency>
        <groupId>com.graphql-java</groupId>
        <artifactId>graphql-java</artifactId>
        <version>8.0</version>
    </dependency>

    <dependency>
        <groupId>com.graphql-java</groupId>
        <artifactId>graphql-java-tools</artifactId>
        <version>3.2.0</version>
    </dependency>

</dependencies>
```
## 💎 Step 2 - Define GraphQL Schema

Next step is to write the schema in article.graphql at the root of the project (default, can be changed):

We have three data types: Article, Author, Comment and one input: ArticleInput.

The other scalar types that were used(ID, String) are already defined in GraphQL Specification Schema.

```graphql 
schema {
    query: Query
    mutation: Mutation
}

type Query {
       hello(message:String): String
       articleById(id: String): Article
       articles(filter: Filter):[Article]
}

type Mutation {
    createArticle(article: ArticleInput): Article!
}

input ArticleInput {
    id: ID
    title: String
    tags: [String]
    content: String
    creationDate: String
    readingTime: Int
    image: String
}

type Article {
    id: ID!
    title: String!
    tags: [String]
    content: String
    author: Author
    creationDate: String
    lastModified: String
    readingTime: Int
    image: String
    comments: [Comment]
}

type Author {
    id: ID!
    firstName: String
    lastName: String
    articles(count: Int): [Article]
    contactLinks: [String]
}

type Comment {
    id: ID!
    text: String
    commentAuthor: String
}

input Filter {
    tags: [String]
    firstName: String
    lastName: String
    startDate: String
    endDate: String
    title: String
}
```

## 🎬 Step 3 - Initialize GraphQL

The responsibility of this class would be to initialize GraphQL. This is done by first parsing the above schema file and then the resolvers.

Call .build() and .makeExecutableSchema() to get a graphql-java GraphQLSchema.
```java
@Component
public class GraphInit {

    private GraphQLSchema buildSchema(ArticleRepository articleRepository, AuthorRepository authorRepository, CommentRepository commentRepository) throws IOException {

        return SchemaParser.newParser()
                .file("article.graphql")
                .resolvers(
                        new Query(articleRepository),
                        new Mutation(articleRepository),
                        new ArticleResolver(authorRepository, commentRepository),
                        new AuthorResolver(articleRepository)
                )
                .build()
                .makeExecutableSchema();
    }

    @Bean
    public GraphQL graphQL(ArticleRepository articleRepository, AuthorRepository authorRepository, CommentRepository commentRepository) throws IOException {

        GraphQLSchema graphQLSchema = buildSchema(articleRepository, authorRepository, commentRepository);

        return GraphQL.newGraphQL(graphQLSchema)
                .build();
    }

}
```


## 🔑 Step 4 - Resolvers and Data Classes

GraphQL Java Tools maps fields on your GraphQL objects to methods and properties on your java objects. For most scalar fields, a POJO with fields and/or getter methods is enough to describe the data to GraphQL. More complex fields (like looking up another object) often need more complex methods with state not provided by the GraphQL context (repositories, connections, etc). GraphQL Java Tools uses the concept of “Data Classes” and “Resolvers” to account for both of these situations.
 
Given the above schema, GraphQL Java Tools will expect to be given six classes that map to the GraphQL types: Query, Mutation, ArticleInput, Article, Author and Comment.

The Data classes for Article, Author and Comment are simple:

```java
 class Article {

    private String id;
    private String title;
    private List<String> tags;
    private String content;
    private Author author;
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd")
    @JsonDeserialize(using = LocalDateDeserializer.class)
    @JsonSerialize(using = LocalDateSerializer.class)
    private LocalDate creationDate;
    private LocalDate lastModified;
    private int readingTime;
    private String image;
    private List<Comment> comments;
    
    // constructor

    // getters and setters
}
```
```java
class Author {

    private String id;
    private String firstName;
    private String lastName;
    private List<Article> articles;
    private List<String> contactLinks;
    
    // constructor

    // getters and setters


}

```

```java
class Comment {

    private String id;
    private String text;
    private String commentAuthor;

    // constructor

    // getters and setters

}
```

The complex fields on Query, Mutation, Article and Author are handled by "Resolvers". Resolvers are object instances that reference the “Data Class” they resolve fields for.

### GraphQLResolver

The ArticleResolver (resolver for the Article "Data Class") look something like this:

```java
public class ArticleResolver implements GraphQLResolver<Article> {

    private final AuthorRepository authorRepository;
    private final CommentRepository commentRepository;

    public ArticleResolver(AuthorRepository authorRepository, CommentRepository commentRepository) {
        this.authorRepository = authorRepository;
        this.commentRepository = commentRepository;
    }

    public CompletableFuture<Author> author(Article article) {
        return CompletableFuture.supplyAsync(() -> {
            return AuthorMapper.entityToGraphQL(authorRepository.getById(article.getAuthor().getId()));
        });
    }

    public List<Comment> comment(Article article) {
        return commentRepository.getComments(article)
                        .stream()
                        .map(CommentMapper::entityToGraphQL)
                        .collect(Collectors.toList());
    }


}
```

When given a ArticleResolver instance, GraphQL Java Tools first attempts to map fields to methods on the resolver before mapping them to fields or methods on the data class. If there is a matching method on the resolver, the data class instance is passed as the first argument to the resolver function. 

### Root Resolvers

Since the Query/Mutation objects are root GraphQL objects, they does not have an associated data class. In those cases, any resolvers implementing GraphQLQueryResolver or GraphQLMutationResolver will be searched for methods that map to fields in their respective root types. Root resolver methods can be spread between multiple resolvers, but a simple example is below:

#### Define GraphQL Queries

```java
class Query implements GraphQLQueryResolver {

    private final ArticleRepository articleRepository;

    public Query(ArticleRepository articleRepository) {
        this.articleRepository = articleRepository;
    }

    public Article articleById(String id) {
           return ArticleMapper.entityToGraphQL(articleRepository.getById(id));
    }

}
```
#### Define GraphQL Mutations

```java
class Mutation implements GraphQLMutationResolver {

    private final ArticleRepository articleRepository;

    public Mutation(ArticleRepository articleRepository) {
        this.articleRepository = articleRepository;
    }

    public Article createArticle(ArticleInput article) throws IOException {
        return ArticleMapper.entityToGraphQL(articleRepository.saveArticle(article));
    }
}
```

Resolvers must be provided to the schema parser. 

### GraphQL query samples

The request URL is: `http://localhost:8080/api/graph`

-Get all articles that contain given tags:
- GraphQL
```graphql 
query articlesByTag($tags: [String]){
       articles(filter: {
          tags: $tags
       })
       
      {
           title
           content
           tags
           author {
               firstName
               articles(count:1) {
               title
               }
               contactLinks
           }
           comments {
               commentAuthor
               text
           }
          
       }
}
```

- GraphQL Variables
```json5
{
	"tags":["Architecture"]
}
```

- Response
```json5
{
    "data": {
        "articles": [
            {
                "title": "Best Practices for REST API Error Handling",
                "content": "REST is a stateless architecture in which clients can access and manipulate resources on a server.",
                "tags": [
                    "Architecture",
                    "REST"
                ],
                "author": {
                    "firstName": "Michael",
                    "articles": [
                        {
                            "title": "Best Practices for REST API Error Handling"
                        }
                    ],
                    "contactLinks": [
                        "GitHub"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Ion Popescu",
                        "text": "First comment"
                    }
                ]
            }
        ]
    }
}
```

-Ger all articles from a specific author

- GraphQL

```graphql 
query articlesByAuthor($firstName: String, $lastName: String){
    articles(filter: {
         firstName: $firstName, lastName: $lastName
      })
      
    {
          title
          content
          tags
          author {
              firstName
              articles(count:1) {
              title
              }
              contactLinks
          }
          comments {
              commentAuthor
              text
          }
         
    }
}
```
- GraphQL Variable
```json5
{
	"firstName": "Justin",
	"lastName": "Albano"
}
```
- Response
```json5
{
    "data": {
        "articles": [
            {
                "title": "Causes and Avoidance of java.lang.VerifyError",
                "content": "In this tutorial, we'll look at the cause of java.lang.VerifyError errors and multiple ways to avoid it.",
                "tags": [
                    "Java",
                    "JVM"
                ],
                "author": {
                    "firstName": "Justin",
                    "articles": [
                        {
                            "title": "Causes and Avoidance of java.lang.VerifyError"
                        }
                    ],
                    "contactLinks": [
                        "GitHub",
                        "Twitter"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Gigel",
                        "text": "Second comment"
                    }
                ]
            },
            {
                "title": "A Guide to Java HashMap",
                "content": "Let's first look at what it means that HashMap is a map. A map is a key-value mapping, which means that every key is mapped to exactly one value and that we can use the key to retrieve the corresponding value from a map.",
                "tags": [
                    "Java"
                ],
                "author": {
                    "firstName": "Justin",
                    "articles": [
                        {
                            "title": "Causes and Avoidance of java.lang.VerifyError"
                        }
                    ],
                    "contactLinks": [
                        "GitHub",
                        "Twitter"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Admin",
                        "text": "There are no comments yet."
                    }
                ]
            }
        ]
    }
}

```
-Get all articles between two dates
- GraphQL
```graphql 
query articlesByDate($startDate: String, $endDate: String){
   articles (filter: {
           startDate: $startDate, endDate: $endDate
       })
       
      {
           title
           content
           tags
           author {
               firstName
               articles(count:1) {
               title
               }
               contactLinks
           }
           comments {
               commentAuthor
               text
           }
          
       }
}

```
- GraphQL Variables
```json5
{
    "startDate": "2019-01-01",
    "endDate": "2019-03-01"
}
```
- Result
```json5
{
    "data": {
        "articles": [
            {
                "title": "Causes and Avoidance of java.lang.VerifyError",
                "content": "In this tutorial, we'll look at the cause of java.lang.VerifyError errors and multiple ways to avoid it.",
                "tags": [
                    "Java",
                    "JVM"
                ],
                "author": {
                    "firstName": "Justin",
                    "articles": [
                        {
                            "title": "Causes and Avoidance of java.lang.VerifyError"
                        }
                    ],
                    "contactLinks": [
                        "GitHub",
                        "Twitter"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Gigel",
                        "text": "Second comment"
                    }
                ]
            }
        ]
    }
}

```
-Get all articles
- GraphQL
```graphql 
query articles{
    articles {
           title
           content
           tags
           author {
               firstName
               articles(count:1) {
               title
               }
               contactLinks
           }
           comments {
               commentAuthor
               text
           }
          
       }
}

```
- Result
```json5
{
    "data": {
        "articles": [
            {
                "title": "Best Practices for REST API Error Handling",
                "content": "REST is a stateless architecture in which clients can access and manipulate resources on a server.",
                "tags": [
                    "Architecture",
                    "REST"
                ],
                "author": {
                    "firstName": "Michael",
                    "articles": [
                        {
                            "title": "Best Practices for REST API Error Handling"
                        }
                    ],
                    "contactLinks": [
                        "GitHub"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Ion Popescu",
                        "text": "First comment"
                    }
                ]
            },
            {
                "title": "Causes and Avoidance of java.lang.VerifyError",
                "content": "In this tutorial, we'll look at the cause of java.lang.VerifyError errors and multiple ways to avoid it.",
                "tags": [
                    "Java",
                    "JVM"
                ],
                "author": {
                    "firstName": "Justin",
                    "articles": [
                        {
                            "title": "Causes and Avoidance of java.lang.VerifyError"
                        }
                    ],
                    "contactLinks": [
                        "GitHub",
                        "Twitter"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Gigel",
                        "text": "Second comment"
                    }
                ]
            },
            {
                "title": "A Guide to Java HashMap",
                "content": "Let's first look at what it means that HashMap is a map. A map is a key-value mapping, which means that every key is mapped to exactly one value and that we can use the key to retrieve the corresponding value from a map.",
                "tags": [
                    "Java"
                ],
                "author": {
                    "firstName": "Justin",
                    "articles": [
                        {
                            "title": "Causes and Avoidance of java.lang.VerifyError"
                        }
                    ],
                    "contactLinks": [
                        "GitHub",
                        "Twitter"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Admin",
                        "text": "There are no comments yet."
                    }
                ]
            }
        ]
    }
}

```
-Get article by id
- GraphQL
```graphql 
query articlesById($id: String){
   articleById(id: $id){
       ...ArticleFragment
   }
}

fragment ArticleFragment on Article {
  title 
  tags 
  content
  author{
      firstName
      articles(count:2) {
          title
      }
      contactLinks
  }
  creationDate
  lastModified
  readingTime
  image
  comments {
      text
      commentAuthor
  }
}
```
- GraphQL Variables
```graphql 
{
	"id": "1"
}
```
- Result
```json5
{
    "data": {
        "articleById": {
            "title": "Best Practices for REST API Error Handling",
            "tags": [
                "Architecture",
                "REST"
            ],
            "content": "REST is a stateless architecture in which clients can access and manipulate resources on a server.",
            "author": {
                "firstName": "Michael",
                "articles": [
                    {
                        "title": "Best Practices for REST API Error Handling"
                    }
                ],
                "contactLinks": [
                    "GitHub"
                ]
            },
            "creationDate": "2018-10-22",
            "lastModified": "2019-01-11",
            "readingTime": 3,
            "image": "https://res.cloudinary.com/fittco/image/upload/w_1920,f_auto/ky8jdsfofdkpolpac2yw.jpg",
            "comments": [
                {
                    "text": "First comment",
                    "commentAuthor": "Ion Popescu"
                }
            ]
        }
    }
}
```
-Get articles that contain a specific word in title
- GraphQL
```graphql 
query allArticlesByTitle($title: String){
   articles(filter: {
         title: $title
      })
      
     {
          title
          content
          tags
          author {
              firstName
              articles(count:1) {
              title
              }
              contactLinks
          }
          comments {
              commentAuthor
              text
          }
         
      }
}
```
- GraphQL Variables
```json5
{
	"title": " java"
}
```
- Result
```json5
{
    "data": {
        "articles": [
            {
                "title": "Causes and Avoidance of java.lang.VerifyError",
                "content": "In this tutorial, we'll look at the cause of java.lang.VerifyError errors and multiple ways to avoid it.",
                "tags": [
                    "Java",
                    "JVM"
                ],
                "author": {
                    "firstName": "Justin",
                    "articles": [
                        {
                            "title": "Causes and Avoidance of java.lang.VerifyError"
                        }
                    ],
                    "contactLinks": [
                        "GitHub",
                        "Twitter"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Gigel",
                        "text": "Second comment"
                    }
                ]
            },
            {
                "title": "A Guide to Java HashMap",
                "content": "Let's first look at what it means that HashMap is a map. A map is a key-value mapping, which means that every key is mapped to exactly one value and that we can use the key to retrieve the corresponding value from a map.",
                "tags": [
                    "Java"
                ],
                "author": {
                    "firstName": "Justin",
                    "articles": [
                        {
                            "title": "Causes and Avoidance of java.lang.VerifyError"
                        }
                    ],
                    "contactLinks": [
                        "GitHub",
                        "Twitter"
                    ]
                },
                "comments": [
                    {
                        "commentAuthor": "Admin",
                        "text": "There are no comments yet."
                    }
                ]
            }
        ]
    }
} 

```
-Create new article
- GraphQL
```graphql 
mutation createArticle($article: ArticleInput){
   createArticle(article: $article){
       ...ArticleFragment
   }
}

fragment ArticleFragment on Article {
  id
  title
  tags
  content
  creationDate
  readingTime
  image
  
}
```
- GraphQL Variables
```graphql 
{
	"article": {
		"title": "How to Read HTTP Headers in Spring REST Controllers",
		"tags": ["Rest", "Spring Web"],
		"content": "IIn this quick tutorial, we're going to look at how to access HTTP Headers in a Spring Rest Controller.",
		"creationDate": "2020-03-03",
		"readingTime": 6,
		"image": "https://res.cloudinary.com/fittco/image/upload/w_1920,f_auto/ky8jdsfofdkpolpac2yw.jpg"
	}
}
```
- Result
```graphql 
{
    "data": {
        "createArticle": {
            "id": "266000310",
            "title": "How to Read HTTP Headers in Spring REST Controllers",
            "tags": [
                "Rest",
                "Spring Web"
            ],
            "content": "IIn this quick tutorial, we're going to look at how to access HTTP Headers in a Spring Rest Controller.",
            "creationDate": "2020-03-06",
            "readingTime": 6,
            "image": "https://res.cloudinary.com/fittco/image/upload/w_1920,f_auto/ky8jdsfofdkpolpac2yw.jpg"
        }
    }
}
```

## 🔗 Helpful links

- https://www.graphql-java-kickstart.com/tools/getting-started/
- https://www.graphql-java.com/documentation/v11/data-fetching/
- https://www.baeldung.com/spring-graphql
- https://graphql.org/learn/queries/
- https://www.youtube.com/watch?v=oPZoNjyTW3w&list=PL3bZ4yI4IY1DsPoN1Cilz79kxFdeiLT_O
- https://swapi.co/
- https://swapi.graph.cool/
- https://github.com/APIs-guru/graphql-apis