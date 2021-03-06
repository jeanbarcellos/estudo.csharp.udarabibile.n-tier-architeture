_Repositório somente para estudos!_

# N-Tier Architecture in ASP.NET Core

Part 4: Build layered architecture for separation of concern, scalability and maintenance

Instrutor:

- [Udara Biblie](https://github.com/udarabibile)

Referências:

- https://chathuranga94.medium.com/n-tier-architecture-in-asp-net-core-d1f1b14f2010
- https://github.com/udarabibile/aspnetcore-webapi/tree/n-tier-architecture

<br>
<br>
<hr>

**Criação do diretório**

```
mkdir webapi

cd webapi
```

**Abrir o VS Code**

```bash
code .
```

**Abrir e criar o diretório `src/` (onde ficarão os projetos)**

```bash
mkdir src
```

**Criar solução para arquitetura N-Tier**

```
dotnet new sln --name webapi
```

## Create Data Access Layer (DAL)

```bash
// Create class library and add this project to solution
dotnet new classlib --name webapi.data --output src/webapi.data
dotnet sln add src/webapi.data/webapi.data.csproj

cd src/webapi.data

// Navigate to DAL → cd webapi.data and Install Nuget packages
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.InMemory

cd ../..
```

## Create Business Logic Layer (BLL)

```bash
// Create class library and add this project to solution
dotnet new classlib --name webapi.business --output src/webapi.business
dotnet sln add src/webapi.business/webapi.business.csproj

cd src/webapi.business

// Navigate to BLL → cd webapi.business and reference to DAL project
dotnet add reference ../webapi.data/webapi.data.csproj

cd ../..
```

## Create Presentation Layer (UIL)

```bash
// Create Web API project and add this project to solution
dotnet new webapi --name webapi.api --output src/webapi.api
dotnet sln add src/webapi.api/webapi.api.csproj

cd src/webapi.api

// Navigate to UI → cd webapi.api and reference to BL project
dotnet add reference ../webapi.business/webapi.business.csproj

cd ../..
```

<br>
<hr>

E quanto a entidades comuns, como modelos?

Normalmente, de acordo com o código anterior, os modelos eram mais acoplados aos _repositórios_. Assim, os _modelos_ podem ser adicionados à camada de acesso a dados (DAL), mas os _modelos_ foram usados ​​em _serviços_ e _controladores_ para validações ou como tipos de retorno. Isso não seria um problema para a camada de lógica de negócios (BLL), pois já estaria se referindo à camada de acesso a dados (DAL).

No entanto, para usar modelos em controladores, **a camada de apresentação (UIL) deve referenciar a camada de acesso aos dados (DAL) diretamente e isso seria impróprio para o projeto**. A referência de `webapi.api → webapi.data` ignoraria `webapi.business` e também herdaria todos os _repositórios_, exceto os _modelos_, portanto, não pode ser considerada a melhor prática. (Observe que pode ser validado onde os modelos são usados ​​apenas e os repositórios não são alterados, mas isso é falha de design.)

Assim, é adicionada a camada de infraestrutura chamada `webapi.core`, com _entidades_ comuns. Aqui, ele pode incluir normalmente **modelos, perfilagem e todas as outras camadas que fazem referência e usam essas entidades comuns**.

Desse modo, **a camada de suporte pode ser criada para a infraestrutura central**, como modelos de domínio, registrador onde todas as outras 3 camadas estariam referenciando.

<br>
<br>

## Create Infrastructure Layer (Core)

As entidades comuns para o aplicativo são armazenadas em _Models_ e a biblioteca de classes é criada para essas alterações principais. Todas as outras camadas farão referência à camada central e importarão modelos para uso.

```bash
// Create class library and add this project to solution
dotnet new classlib --name webapi.core --output src/webapi.core
dotnet sln add src/webapi.core/webapi.core.csproj

// Navigate to each of DAL, BLL, UI projects and refer Core project
dotnet add reference ../webapi.core/webapi.core.csproj
```

<br>
<hr>

## Configurar a injeção de dependência e dependência circular

Até este momento, todas as dependências foram injetadas `StartUp.cs`, onde a implementação relevante foi mapeada para as abstrações necessárias. Por exemplo, o concreto `AuthorServicefoi` mapeado para `IAuthorService` ser injetado. No entanto, resolver a dependência de `IRepository` em `webapi.api` resultaria na adição de uma referência a `webapi.data → webapi.api`, violando assim as restrições de fluxo de dados. Aqui, **a camada de apresentação (UIL) dependerá diretamente da camada de acesso aos dados(DAL)**.

Conseqüentemente, a injeção de dependência deveria ser feita em um local separado e tal camada poderia ser `webapi.core`. Mas, a partir de agora, existem referências a `webapi.core` camadas de outras camadas e, com esse requisito, `webapi.core` deve-se fazer referência a outras camadas para resolver dependências. Simplesmente dependências seriam `webapi.api ↔ webapi.core` e, portanto, **dependência circular** . _Nenhuma das camadas pode ser compilada_, pois ambas dependem uma da outra. (Mesmo que o link seja para finalidades diferentes, há referência entre as camadas). Assim, `webapi.core` acaba por não ser adequado para injeção de dependência.

**Composition Root**

Local único onde as dependências são registradas..

**Nota**: pode haver design onde a raiz da composição pode ser evitada. Camadas como DAL → Repositórios, BL → Serviços, UI → Controladores, App → WebAPI + DI

<br>

### Create Dependency Injection Layer (Root)

**A resolução das dependências é feita no local central** e passada para o contêiner Inverso de Controle integrado para a injeção de dependência.

```bash
// Create class library and add this project to solution
dotnet new classlib --name webapi.root --output src/webapi.root
dotnet sln add src/webapi.root/webapi.root.csproj

// Navigate to composition root and Install Nuget packages
dotnet add package Microsoft.Extensions.DependencyInjection.Abstractions
dotnet add package Microsoft.EntityFrameworkCore.InMemory

// Add reference to DAL, BL projects
dotnet add reference ../webapi.business/webapi.business.csproj
dotnet add reference ../webapi.data/webapi.data.csproj
```

Injeção de dependência embutido no ASP.NET Core é feito em `StartUp.cs` de `webapi.api`, assim ele vai se referir `webapi.root` para iniciar a resolução de dependência.

```bash
// Navigate to webapi.api and refer webapi.root project
dotnet add reference ../webapi.root/webapi.root.csproj
```

E use a classe `CompositionRoot` para iniciar a resolução de dependências

```bash
public void ConfigureServices(IServiceCollection services)
{
   CompositionRoot.injectDependencies(services);
   services.AddControllers();
}
```

<br>
<br>
<hr>

**Data Flow: Presentation → Business Logic → Data Access**

Vamos considerar o fluxo de dados por meio de **controllers**, **services** e **repositories** `GetAuthorByName` do endpoint REST ao banco de dados:

- Controllers (Presentation Layer) → Services (Business Logic Layer)

```cs
// AuthorController
[HttpGet("{authorName}")]
public Task<Author> GetAuthorByName(String authorName) =>
   authorService.GetAuthorByName(authorName);
```

- Services (Business Logic Layer) → Repositories (Data Access Layer)

```cs
// AuthorService
public Task<Author> GetAuthorByName(string firstName) =>
   _unitOfWork.AuthorRepository.GetByName(firstName);
```

- Repositories (Data Access Layer) → Database Context (Database)

```cs
// AuthorRepository
public Task<Author> GetByName(string name) =>
   context.Set<Author>().FirstOrDefaultAsync(a => a.Name == name);
```

<br>
<br>
---

## Executar

Acessar a raiz do projeto

**Rodar com ouvinte:**

```
dotnet watch --project src/webapi.api run
```

**Apenas rodar**

```
dotnet run --project src/webapi.api
```

<br>
<br>
---

## Além dos comandos fornecidos acima, alguns são úteis no caso de pequenas alterações

Remove reference from one project to another

```bash
dotnet remove reference ../webapi.core/webapi.core.csproj
```

Remove project from solution

```bash
dotnet sln remove webapi.core/webapi.core.csproj
```

Remove Nuget package from project

```bash
dotnet remove package
```

Clean .NET Core solution or project if references are cached

```bash
dotnet clean
```

Publish ASP.NET Core Web API into given MacOSX version

```bash
dotnet publish aspnetcore.sln -c Release -r osx.10.14-x64 --output ./build --framework netcoreapp5.0
```

Run built assemblies in given OS

```bash
dotnet build/webapi.api.dll
```

<br>
<br>
---

## Design (References)

![Design](data:image/png;base64,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)
