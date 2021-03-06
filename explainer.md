# Web Shared Libraries
//TODO: Figure out header stuff
//November, 2019
//Status: Draft

Web Shared Libraries attempts to improve website loading by sharing cache across sites. This document describes the opportunity and challenges involved, along with techniques to address the various concerns and metrics to evaluate impact.

### Background

Long load times introduce costs to both users and publishers, and can significantly reduce a site�s retention, conversion, and effectiveness. Reducing load time is a significant challenge to web developers, and this challenge increases as sites become richer and more complex.

Caching resources is a popular technique for reducing load-time, with obvious improvements after the initial load. To address the obvious privacy [[1](https://old.iseclab.org/papers/ccn-cache-attacks-iseclab-0812-001.pdf)] implications,  [doubled-keyed caching](https://github.com/whatwg/fetch/issues/904) has been adopted. This proposal preserves the privacy model of double-key caching while improving cache hit rates by pre-loading popular content into a cache that is shared across all browser-contexts.

## Proposal

Define a set of common assets (libraries, fonts, etc) that are in common use. This is called the Cache-Candidate-List, and is generated out of band. See [Signals](#Signals) for a discussion of the selection criteria and [Governance](#Governance) for a discussion of the processes involved.

Browsers then pre-load all or some of the Cache-Candidate-List.

> There was a suggestion to allow browsers to add assets into the common cache based on user navigation. (IE: references to common libraries get loaded then promoted into the common cache) 
**Note**: Mozilla believes that the privacy attack surface here is complex and difficult to model, and thus that we should avoid introducing dependencies between the performance characteristics of site X and previous visits to site Y. In other words, Mozilla favors a precache-only approach.

> [Delete?] Sharing a single cache across multiple load contexts introduces some obvious privacy risks. Specifically, sites could infer information about a visitor�s web history due to the presence or absence of files in the cache. This proposal contains adaptations to mitigate these risks. 

## Challenges / Risk Model(s)

### I: Privacy

Sharing cache contents trivially allows a site to detect if the user had previously visited another site. [Add footnote here: See the attacks documented in [slides 11 and 12 of the BlinkOn deck](https://docs.google.com/presentation/d/1CTC_BLpBqLf7B82b1ytIm0l84SjK45bk7UaCyRpyu5A/edit#slide=id.g73eb2e1894_0_20)] 

Pre-loading files into the cache before any other sites access the libraries *eliminates* this risk. IE: Cache contents are not based on the user's actions/navigations.

We have also discussed strategies for adding new content into the cache (based on user activity) while limiting/reducing the privacy implications:
* Limiting which files can be cached greatly reduces this risk. Specifically, if a library file is in use by n common sites, then an observer could only discover that the user has visited one of n other sites previously. The information leakage (entropy) is greatly reduced for reasonably large values of n. 
IE: only share cache from libraries that are known to be used by a large number of sites.
* Adding files to the cache when they are first used by a site increases this risk, although this effect might be minimized by caching a limited number (one?) of files per site visited.
  * **Note**: Mozilla believes that the privacy attack surface here is complex and difficult to model, and thus that we should avoid introducing dependencies between the performance characteristics of site X and previous visits to site Y. In other words, Mozilla favors a precache-only approach.

### II: Incumbent popularity/Reducing upgrades/Stifling innovation

Pre-cached files will provide a significant advantage to existing versions of popular libraries, and may indirectly cause developers to avoid less popular candidates. This could reduce the frequency of updates, and discourage experimentation among library authors.

In order to minimize this effect, we propose a number of strategies:

* Stochastic Pre-Caching: Create a longer Cache-Candidate-List, and randomly pre-load a subset of items from that list. The selection logic should be weighted so that items near the top of the list are very likely to be pre-loaded, while lower ranked items are increasingly less likely. 
This allows emerging libraries to have a gradually growing chance of being pre-cached, and also combines well with Late-Caching.
  * One reviewer (can�t recall who at the moment - sorry) commented that the sub-list stored for a user might be tested as a pseudo-identifier, especially if each user chooses a random assortment of assets to load from the master list.
  This could be addressed by generating a small number of independent lists and distributing those to the community.
* We could allow library authors to intentionally push new versions into the list (and gradually evict old entries) in order to reduce the disincentive to ship updates. By allowing multiple versions of a library to overlap for some period of time in the cache, we can enable libraries to evolve while also nudging the ecosystem away from wide version proliferation. [footnote: See https://blog.cloudflare.com/javascript-libraries-are-almost-never-updated/]
* The signals used to rank the Cache-Candidate-List could include factors such as growth and usage of previous versions of a given library.

### III: Wasting bandwidth

Pre-caching may result in users downloading files that they never use.
* Consider only downloading on high-bandwidth connections. This may introduce some amount of entropy where sites can identify a user�s bandwidth history.
* Consider progressive/proportional downloads over time. This mitigates up-front costs, but could introduce signal regarding the user�s frequency of browser usage.

## Signals

The Cache-Candidate-List is a ranked list of resources that can potentially be cached. The following factors might be considered when ranking the list.
* *Usage*: Reward libraries that are used in a large number of top 10K sites.
* Popularity (*Delta*): Reward libraries that are increasing in usage month over month.
* Size: Unknown. Very small libraries might not be worth streaming? Very large libraries might not be used (and contribute to wasted bandwidth)?
* Updates: Reward libraries that upgrade, anticipating that release n+1 will eventually get the same usage as release n. 
[Open question: How do we know that two libreary instances are related?]


# Metrics

This list describes a number of metrics that can evaluate the impact of Web Shared Libraries. Note that some of these metrics require us to track which sites have accessed specific cached libraries, which would otherwise not be necessary.
* Download Bytes Saved (Maximum) - Add up all shared cache hits, and report on the sum. Note: this ignores the potential savings from a domain-linked cache.
* Download Bytes Saved (Actual) - If we can track which sites have accessed a given file, actual bytes saved would register once for each site that accesses the file in cache.
* Bytes downloaded for pre-caching. (Negative or cost metric to be minimized.)
* Disk space saved. For cases when a library is shared between domains, the number of bytes saved from domain-specific caches. (Expect that double keyed cache implementations will already optimize for this.)
* Uptake. Assuming that the spec requires developers to include checksums in their file references, we can track the proportion of file references that include a checksum. This might be tuned to only count references that should include a checksum.

# Open Questions

## I: Governance
* Who determines the ranking system for the Cache-Candidate-List? 
* Where is the list stored, accessed? 
* What�s the frequency of update?

## II: Participation Mechanism for Library/Framework Authors
* Is there a mechanism for library authors to interact with the Cache-Candidate-List and/or the selection criteria? It might be helpful if authors could submit a new update for an existing library, and thereby gain the caching benefit from the current popularity.
  * This avoids a potential <I�ll monkey-fit a small library that loads my previous popular version and make minimal mods to it on the fly so I don�t lose my position in the cache> pattern.
  * It also allows us to combat version proliferation in the ecosystem by actively managing which versions of a library may be cached.

## III: Tooling (AKA WebPack)

Many sites use tools such as WebPack to compress libraries and reduce the number of loaded files. 
* Consider reviewing assets currently in use to determine how often this happens, and evaluate what �flattening� these into components might gain.
* Consider adapting tools (WebPack, etc) to prefer common references.
* Consider adopting those tools to include identity checksums.

### TODO/Notes:
* Submit/upload
* Discuss impact (current and expected, based on bundling and uptake of checksums)
* Reference: flackr�s [blinkon deck](https://docs.google.com/presentation/d/1CTC_BLpBqLf7B82b1ytIm0l84SjK45bk7UaCyRpyu5A/edit)

# Contributors
* November 2019 (early draft) 
girard@chromium.org (author); flackr@chromium.org (author)
vsekhar@chromium.org (reviewer); kinuko@chromium.org (reviewer)
* March 2020 (proposal) 
girard@chromium.org (author); flackr@chromium.org (author)
bholley@mozilla.com (reviewer); vsekhar@chromium.org (reviewer)
<add your name here (as author, reviewer, etc);>

# Appendix: Resolving file references to cache contents

The cache is used when the user agent is confident that a file reference matches an item in the cache. This is normally done by directly matching URL's, and ensuring that the source file's cache restrictions are satisfied. As we move into a shared cache, there are many opportunities to improve the cache hit-rate through the use of integrity checksums (either directly coded by developers or inferred through the use of common locations.)

## Integrity Checksums

If a file references includes an [integrity checksum](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) it can be trivially matched against cache contents. 

* Javascript / CSS
`<link rel=�script|style� href=�...� integrity=�...�>`
* Javascript import (using [module-attributes proposal](https://github.com/tc39/proposal-module-attributes/blob/master/README.md)): 
`import foo from �./foo.js� with integrity: �...�`
* WebAssembly:
`WebAssembly.instantiateStreaming(fetch(�...�, {integrity: �...�}));`
* Unknown/TBD: Fonts, Images

## Allowing cache hits without integrity attribute

For known popular <url, hash> tuples, we can prefetch and serve those resources from the single keyed cache (to the extent the cache headers allow) which benefits loading common resources which are not currently using integrity checksums.

Note that shared caching will work for assets even when the loading page doesn�t use an integrity checksum. Specifically, we observe that some common assets are hosted at well-known location(s) with specific cache directives, and that these assets can be served from the common cache as long as they map to the same checksum within the timespan of the host�s caching rules.

In terms of implementation, when building the Cache-Candidate-List each asset�s checksum is recorded along with the location and cache directions. In cases where assets are in common use (possibly across multiple URLs) we can create a list mapping from URL to checksum. This mapping allows a single cache entry to satisfy each listed URL.

<TODO: Document frequency of this based on flackr�s analysis of top1M sites>. 
