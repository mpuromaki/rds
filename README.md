:warning: Just a friendly reminder that this document is thoughts of a single person. I've started some discussions about this and it seems that there might be fundamental issues both with this approach and directory services for Reticulum in general. I haven't yet been able to think through all of the responses. When that's done, I'll update / archive / delete this document.

# rds - Reticulum Directory Services
Reticulum Directory Services is proposal for name to identity mapping system which fits the zen of reticulum.

And that being said, let's start with an caveat. Zen of Reticulum clearly states "Death to the address". And here I am trying to set up an human readable adressing system for it.
But I strongly feel that having some way to map human readable addresses into identities and destinations could be beneficial to Reticulum. Even in local community networks there could be an need to tell an user over voice channels to connect to an specific resource or system.
Also even if Reticulum isn't created with globalism in mind, there could be some services that become de-facto global standards. These could "own" some easily spellable name without one organization forcing that to everyone.
RDS is meant to follow the basic tenets of not trusting the network and avoiding central systems. Everyone can and must choose who they trust and RDS is just there to help find others in some specific cases.

:information_source: No AI/LLM's were used to write this. I've "discussed" this approach to identity resolution with language models and used them to summarize my own thoughts back to me. I claim responsibility of this concept, this document.

## High level Concept

The idea of RDS is to not follow any current DNS system. Those do not fit the way Reticulum works. Instead of global coordination, every user chooses what RDS server they trust.
Services tell their "home" RDS server what services they provide and users ask their trusted RDS servers where some service can be found.
RDS servers peer just enough to know what service is homed where; more detailed information is requested and cached only if there is an actual request to find that.

This means that no RDS is special based on the technology. There is no tree for anyone to be at the top of. If some RDS server misbehaves, services and users will just move somewhere else. And even then cryptography is used to keep the misbehaving in check.
But RDS servers have some responsibility. This kind of decentralization means that there will be name clashes. RDS server can choose to promote one specific service to "own" the name that everyone is trying to claim. And every RDS server does this choice by themselves.
For services homed in that specific RDS server the choice is clear. RDS server for community will of course give the names for services hosted within that community.

### Some technical concepts

For the examples below, think of an URL like this: ```<protocol>://<organization>:<aspects>```.
This is the problem RDS is trying to solve. How to map that kind of address to something that Reticulum can actually connect to.

Protocol part is simple, it just must be something that the client already knows about. For example "lxmf".

Organization and Aspects are tightly linked to Reticulum identities and aspects, but are used in specific way in RDS. The idea is that is publishing their services over RDS likely run multiple nodes.
These nodes then provide specific services via specific well-known aspects, like ```nomadnet.node``` which is used for announcing nomadnet host.
Very likely there could be multiple independent devices (Reticulum identities) hosting that same NomadNet content, which means there are multiple destinations to get to essentially the same place.

With an example of ```https://rns.recipes``` site over Reticulum RDS it could look something like this: ```nomad:://rns.recipes:nomadnet.node```. (Don't hang up on the details of that URI, it's just for example purposes)  
Where ```rns.recipes``` would just be an name that this "organization" wants to claim to themselves.

Now trying to access that resource would lead to the user calling out to it's trusted RDS whether it knows about that organization and what identities provide those aspects.
This could lead to cache miss, meaning that RDS server would have to request the information from the home RDS server of ```rns.recipes```. And cache it for subsequent queries.

Finally the RDS query from the user would end up in a response containing an organizational identity (Reticulum identity, maybe even network identity) owning that name, aspect that was requested and Reticulum identities that provide that aspect.
Cryptography and signing would have been used to verify that this organization has chosen to use that specific RDS server as home and that those Reticulum identities are something that provide those aspects for this organizational identity.

#### Name collisions

Since name collisions are likely, the organization part could be extended with ```#<characters from encoded identity>``` to select one specific claim of that organization name. That way user could explicitly request specific organization and their services.
On the other hand trusted RDS server could do this implicitly by stating itself that conflicting name ```rns.recipes``` should be one specific organization identity, as a result of consensus within the community.
But also if there is no claim made and multiple organization possibilities exists then it's up to the user to choose specifically where to connect.

Trust on first use (TOFU) can also be utilized here to gain some additional assurances. After the first resolve of name-to-organization the user's client software could cache that link and warn the user if RDS has changed their response.
Something like this should basically instruct the user to question whether it trusts that RDS server any more after that.

## Use cases

There are couple different parts to this RDS system as a whole:
1. How organizations publish via RDS
2. How RDS servers peer with each other
3. How users request records from RDS

### How services publish via RDS

First the organization needs to create identity pair that acts as the owner of the organizational name. This is kind of root of trust, as this identity can then be used to vouch for other reticulum identities that they're approved providers of specific aspects.

Then the organization needs to have selected an RDS they trust. This is social choice and very likely out-of-band. It could be hosted by the organization itself, or some local community for example. It could match the network identity that is providing connectivity.

With this setup the organization creates ```RDS Home``` record, which contains identity of the chosen RDS home server, chosen name of the organization, identity of the organization and signature over those. This record is then sent to the RDS home server, so that they can start peering that information to other RDS servers. And of course RDS server doesn't have to accept that record, instead there would be some out-of-band method for the RDS server to become home of that specific organization identity with that specific name. RDS server would also validate signatures etc of that home record.

Finally the organization could send additional ```RDS Service``` records, which contain Reticulum identities and what aspects they provide, signed by this organization identity.

With this information, this RDS server is ready to act as a home for this organization.

### How RDS servers peer with each other

So at this point we have multiple RDS servers available within the network, with some of them working as home servers for some organizations. But they do not have knowledge of each other, and especially no knowledge of those organizations.

First these RDS servers send announces with aspect ```rds.server``` to the Reticulum network. Respecting Reticulum's guidance on announce frequencies of course. This way over time RDS servers will gather knowledge of other RDS servers. Likely that announce's app data should contain some contact information for the administrator of that server, just so that RDS servers can communicate if necessary.

Then RDS servers acting as a home start sending those ```RDS Home``` records to other RDS servers. I've not yet chosen how to do this, likely over lxmf somehow. Since those records already contain all of the relevant information as a signed packet, this allows RDS servers to start gathering information on where to find details about specific organizations if need arises. Those records contain the organization name and it's identity, so name collisions will become apparent at this point. How to manage those name collisions is largely social problem which can be, at least partially, handled between RDS server operators.

Having RDS servers at all, and having them peer with each other is necessary for couple of reasons. The first reason is to limit bandwidth requirement of this name-to-identity mapping system. The home records could be part of normal announcements to every user, but that would waste so much bandwidth and storage from users that are unlikely to need that information. Secondly it makes the user experience much better: since the RDS servers have been constantly listening for announces over long period of time, they have much better view of what's available and where to find the services.

Now there's a technical detail I've not yet solved: How to decide what home record send to where. Likely there are some algorithms which can make better use of bandwidth than just flooding every record to every known RDS server. Especially when the situation stabilizes and new Home records become less frequent.

### How users request records from RDS

As with the servers, the users themselves need to already trust some RDS server they want to send requests to. This could be an RDS server owned by the same network identity that owns some local transport nodes for example.

The of course the user needs to have a situation where it knows about service it wants to access (like nomadnet.node for rns.recipes), but which it doesn't know the identity or destinations of. So the user sends ```RDS Services Request``` to the trusted RDS.
From this point there are two options. Either the trusted RDS already has the information, or it doesn't.

#### Cache hit, RDS has the relevant records

In this case the trusted RDS server already knows the requested organization name, knows what organization identity it belongs to and has recent records of identities providing the requested aspects. The RDS server just sends back ```RDS Services Response``` message back to the user which contains all ```RDS Service``` records for identities which provide the requested aspect. There could be none, one or even multiple.

#### Cache miss, RDS doesn't have relevant records

In this case the trusted RDS server doesn't have the relevant records locally cached. If the RDS server doesn't have any ```RDS Home``` records cached for that name either, then it must respond with an error of unknown organization.
But if the trusted RDS Server has at least one home record, it can start working on gathering the required details.

The trusted RDS server sends ```RDS Services Request``` to the RDS home server listed in the home record. The RDS home server responds with ```RDS Services Response```, which this trusted server caches for subsequent requests.
Then this trusted RDS server responds to the user with the information provided in that services response.

Finally this means that the user has Reticulum identities for the requested aspect. This allows the user to calculate Reticulum Destination Hash, which can then be used to connect directly to the service the user was trying to find.
