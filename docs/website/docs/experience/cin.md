# CIN

!!! note ""
    Visit the [official website](https://cinfrance.fr/) (French).

The CIN (Cargo Information Network) is a French company owned by Air France and other airlines and freight actors. It is tasked with ensuring the connectivity of each actor's systems with French customs. CIN is deployed on multiple French airports, most notably the Charles de Gaulle (CDG) airport, the largest French airport.

CIN is critical to allow the movements of goods in and out of France.

## My involvement

I was a software engineer in the CIN team between 2022 and 2024. I was involved with maintenance, the initial implementation of the ANTES module (goods import) and the rewrite of the ECS (Export Control System) module.

I worked on production performance improvements including the implementation of server-side paging, as well as build-time improvements around Gradle and our CI. I worked on improving the reliability of builds by creating internal tools to facilitate testing, decrease caching issues and detecting missing checks in CI.
I worked on unifying the user experience of the many screens and modules.

## Technical stack

- Kotlin microservices with Http4K
- Legacy core in Java with RestX
- TypeScript / Angular frontend
- MongoDB
- HubIT (pub-sub service developed at 4SH)
