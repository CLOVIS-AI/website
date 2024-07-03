# Arcachon town hall

!!! note ""
    Visit the [official website](https://www.ville-arcachon.fr/) (French).

Arcachon is a small beach city in south-west France. Rich from tourism, it lives its peak activity during the summer. During the rest of the year, it is a calm town with a large number of retired inhabitants.

As in any other city, staff from the town hall has to process tons of paperwork, often on actual paper. Some processes are somewhat computerized using Excel and email for communication, which leads to vast amounts of outdated information being shared internally in a way that is hard to back up or control in case of GDPR requests.

Some processes are automated by specific tools that interoperate badly with the city's existing infrastructure (iframes in the main website, or separate websites with their own branding, payment interfaces that do not support HTTPS…).

As the city was preparing for a full rewrite of their official site, it was crucial to possess an integrated system to deal with the different workflows, that would integrate itself directly into the new website, while automating data sharing between employees, with full control from the IT department.

No off-the-shelf tools perform these features while following the policies set by the elected council (e.g. inhabitants must not need to create an account). The town hall's IT department created an internship offer to custom-build this system.

## Formulaide

!!! note ""
    View the [repository](https://gitlab.com/arcachon-ville/formulaide).

The goal was to create an internal tool called Formulaide that would let the IT department declare the structure of their processes: data requested from users, who would be responsible from verifying and processing it, backup policies, etc.

The new website would be developed after Formulaide and would interact with Formulaide's APIs to dynamically insert the required fields into its editor, letting the PR team construct forms as they wanted.

Formulaide would thus receive the user's submissions directly, and would store them, giving visibility rights to relevant staff, creating todo-lists to staff needing to verify or process the data, letting the IT department perform GDPR or backup operations.

## My involvement

As a Software Engineer intern while studying at [ENSEIRB-MATMECA](enseirb.md), I joined the Arcachon town hall to build the project entirely within three months, ahead of the team that would build the new website.

As the only person on the project, it was my job to conduct meeting with the different users, collect their needs, architect the software, develop it, test it and release it.

## Project continuation

At the end of the internship, the town hall contracted me as a freelancer to perform corrective maintenance and implement additional features.

Sadly, the initial context of the internship—during the summer months, when half the staff is on holidays, and the other is extremely busy—meant that very little user feedback could be collected during initial development. After a year in production, much more in-depth user studies, and the start of the development of the new website, a full rewrite was started, but had to be abandoned six months later due to funding issues.

The new website is live as of late 2023, and includes full Formulaide integration, but no Formulaide-based forms are currently publicly available because the development of Formulaide 2.0 was not finished.

## Technical stack (Formulaide 1.0)

- Kotlin backend with Ktor
- Kotlin frontend with React
- Full business logic and validation layers shared between backend and frontend thanks to Kotlin Multiplatform
- MongoDB

## Technical stack (Formulaide 2.0)

Formulaide 2.0 is characterized by its strong preference on versioned immutable entities instead of mutability, allowing to edit forms with no restrictions even after submissions were already submitted.  It also unifies different concepts from the first version, including a complete rethink of form templates.

- Kotlin backend with Ktor
- Kotlin frontend with Compose HTML
- Full business logic and validation layers shared backend and frontend thanks to Kotlin Multiplatform
- Full test suites shared between all app layers
- MongoDB
