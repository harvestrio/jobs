# Harvestr MVP test

## API manipulation

1. Using [Postman](https://www.getpostman.com/), create an Issue on one of your Github Repository
2. Create a Card on a Github Project of your choice

## ES7 understanding

Describe the following Typescript code (Input and Output data models, Code context, detailed description of as many lines as possible )

```ts
export class OrganizationsController {
    static routes(): Router {
        return Router().get("/:organizationId", getOrganization);
    }
}

async function getOrganization(req: AuthedRequest, res: Response) {
    try {
        const id = req.params.organizationId;
        if (
            !(await organizationService.exists(
                isClientId(id) ? { clientId: id } : { id: id },
                req.context
            ))
        )
            throw new Error("Organization not found");
        const organization = <ClientOrganizationView.Fragment>(
            await organizationService
                .getOrganization(isClientId(id) ? { clientId: id } : { id: id })
                .$fragment(CLIENT_ORGANIZATION_VIEW_FRAGMENT)
        );
        const unique_persons = organization.persons;
        const unique_messages = unique_persons
            .flatMap(person => [
                ...person.requested_messages,
                ...person.submitted_messages,
                ...person.cc_messages
            ])
            .filter(
                (message, index, self) =>
                    index === self.findIndex(t => t.id === message.id)
            )
            .map(message => ({
                ...message,
                content: undefined,
                caption:
                    message.content && message.content.length
                        ? message.content
                              .replace(/<[^>]+>/gm, "")
                              .substring(0, 80)
                        : ""
            }));
        const unique_chunks = unique_messages.flatMap(
            message => message.chunks
        );
        const unique_discoveries = unique_chunks
            .flatMap(chunk => chunk.discoveries)
            .filter(
                (discovery, index, self) =>
                    index === self.findIndex(t => t.id === discovery.id)
            );
        res.status(200).send(
            toClientId({
                ...organization,
                persons: unique_persons.map(person => {
                    const {
                        cc_messages,
                        requested_messages,
                        submitted_messages,
                        ...rest
                    } = person;
                    return rest;
                }),
                messages: unique_messages,
                chunks: unique_chunks,
                discoveries: unique_discoveries,
                full: true
            })
        );
    } catch (error) {
        logger.error(error);
        res.status(400).send({ error: error.toString() });
    }
}
```

## Algorithm

Consider an array of n elements, are there two numbers within this array which sum is equal to k?
Ex: [1, 3, 6, -2] returns TRUE if k === -1, FALSE if k === 6.

Write in the language of your choice the corresponding function and give your algorithm's complexity.
Code performance and robustness shall be taken into consideration.

## Code architecture

Describe a possible software architecture (schemas+text) (data models, classes, interactions) for a project with the following specifications:

-   Website fetches user's localization
-   Website displays the list of restaurants around the user
-   For each restaurant, display its distance from user, address, average price, cooking type, and note
-   User can create an account (Username, email, password)
-   Once connected, user can note each nearby restaurant one time (0-5 notation)
-   Restaurant list is accessible through an external Google API

## Coding

1. Deploy and give your Graphql Playground endpoint for the following model using [Prisma](https://www.prisma.io/docs/1.25/get-started/01-setting-up-prisma-new-database-TYPESCRIPT-t002/) (use the Mysql database option):

```ts
type Project {
    id: ID! @unique
    clientId: ID @unique
    createdAt: DateTime!
    updatedAt: DateTime!

    accounts: [Account!]! @relation(name: "ProjectAccounts", onDelete: CASCADE)
    messages: [Message!]! @relation(name: "ProjectMessages", onDelete: CASCADE)
    persons: [Person!]! @relation(name: "ProjectPersons", onDelete: CASCADE)

    name: String! @unique
}

type Account {
    id: ID! @unique
    clientId: ID @unique
    createdAt: DateTime!
    updatedAt: DateTime!
    lastSeenAt: DateTime!
    deletedAt: DateTime

    project: Project! @relation(name: "ProjectAccounts")
    person: Person! @relation(name: "AccountPerson", onDelete: SET_NULL)

    reset_password_token: String
    reset_password_exp_date: DateTime

    username: String! @unique
    hash: String!

    need_onboarding: Boolean! @default(value: "true")
    email_validated: Boolean! @default(value: "false")
    emailConfirmToken: String
}

enum CHANNEL {
    NOTE
    INTERCOM
    MAIL
    SLACK
    ZENDESK
    SHEET
    FORM
}

enum MESSAGE_TYPE {
    NOTE
    MESSAGE
}

type Submessage {
    id: ID! @unique
    clientId: ID @unique
    createdAt: DateTime! # date it was created on harvestr
    updatedAt: DateTime!
    receivedAt: DateTime # real reception date
    message: Message! @relation(name: "MessageSubmessages")

    submitter: Person!

    integration_id: String
    type: MESSAGE_TYPE! @default(value: "MESSAGE")
    content: String! @default(value: "")
}

type Message {
    id: ID! @unique
    clientId: ID @unique
    createdAt: DateTime! # date it was created on harvestr
    updatedAt: DateTime!
    receivedAt: DateTime # real reception date
    _projectId: ID

    project: Project! @relation(name: "ProjectMessages")

    sub_messages: [Submessage!]!
        @relation(name: "MessageSubmessages", onDelete: CASCADE)
    submitter: Person! @relation(name: "MessageSubmitter")
    requester: Person @relation(name: "MessageRequester")
    ccs: [Person!]! @relation(name: "MessageCcs")

    integration_url: String
    integration_id: String
    title: String! @default(value: "")
    content: String! @default(value: "")
    channel: CHANNEL! @default(value: "NOTE")

    read: Boolean! @default(value: "false")
    updated: Boolean! @default(value: "false")
    archived: Boolean! @default(value: "false")
    processed: Boolean! @default(value: "false")
}

enum PERSON_TYPE {
    COLLABORATOR
    CUSTOMER
}

enum RIGHT {
    ADMIN
    AGENT
    VIEWER
}

type ProjectRight {
    project: Project!
    right: RIGHT!

    person: Person @relation(name: "PersonRights")
}

type Person {
    id: ID! @unique
    clientId: ID @unique
    createdAt: DateTime!
    updatedAt: DateTime!
    _projectId: ID

    project: Project! @relation(name: "ProjectPersons")

    right: ProjectRight @relation(name: "PersonRights", onDelete: CASCADE)

    submitted_messages: [Message!]! @relation(name: "MessageSubmitter")
    requested_messages: [Message!]! @relation(name: "MessageRequester")
    cc_messages: [Message!]! @relation(name: "MessageCcs")

    account: Account @relation(name: "AccountPerson")

    deleted: Boolean! @default(value: "false")
    type: PERSON_TYPE! @default(value: "CUSTOMER")
    name: String!
    email: String
    details: String
    phone: String
    zendesk_url: String
}

```

2. Deploy a graphql Typescript server for the previously created model, that allows user to get a Message based on its clientId (no authentication required). 
> Tips: You can use one of the [Prisma example](https://github.com/prisma/prisma-examples) to get started.

## Logic

Consider a turn by turn two players game which goal is to be the first person to reach 50.
Each turn, player must say a number between 1 and 10 (included). This number is added to the total sum.
Would you chose to be the starting player? Why?
