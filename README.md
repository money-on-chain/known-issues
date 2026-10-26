
# Security Audit Report Summary

The Money on Chain ecosystem was analyzed across Oracle Infrastructure, Stablecoin Protocol, Governance, and Dapp security. This document presents a concise summary of identified findings to help other auditors recognize known issues, deduplicate submissions, and avoid repeating completed review work. It intentionally contains only concise root causes, relevant conditions, impacts, evidence, and project dispositions.

## Disposition legend

- **Pending review:** The project has not yet recorded a remediation decision in this summary.
- **Accepted for remediation:** The project recognizes the finding and intends to address it.
- **Accepted and fixed:** The project recognizes the finding and has completed remediation.
- **Rejected — no remediation planned:** The project does not plan to address the finding, without implying a particular reason for that decision.

## Oracle Infrastructure

### OI-01 — Legacy signing addresses may count as separate quorum members
**Affected components:** OMoC-Decentralized-Oracle oracle registration and signature validation.
**Summary:** Updating an oracle signing address can leave historical address-to-owner mappings in storage. Signature deduplication operates on signing addresses rather than resolved owners, so quorum membership is not consistently owner-unique.
**Impact:** Published price or task quorum may contain multiple signatures attributable to one selected owner. Live deployments require storage inspection because a manual cleanup mechanism exists.
**Evidence:** OMoC-Decentralized-Oracle/contracts/OracleManager.sol:114; contracts/libs/IterableOraclesLib.sol:101; contracts/RoundManager.sol:483.
**Disposition:** Accepted and fixed.

### OI-02 — On-chain oracle endpoint metadata becomes unrestricted node network configuration
**Affected components:** OMoC-Decentralized-Oracle registration; OMoC-Node peer communication and IP filtering.
**Summary:** Participant-controlled endpoint strings are consumed for outbound HTTP and DNS-derived inbound authorization without scheme, destination, redirect, resolved-address, or length restrictions.
**Impact:** Node network boundaries and service availability depend on untrusted on-chain metadata, including internal-address reachability and unintended authorization expansion.
**Evidence:** OMoC-Decentralized-Oracle/contracts/Staking.sol:218; contracts/OracleManager.sol:209; OMoC-Node/servers/oracle/src/ip_filter_loop.py:24; servers/common/helpers.py:108.
**Disposition:** Accepted and fixed.

### OI-03 — Synchronous DNS resolution blocks asynchronous node processing
**Affected components:** OMoC-Node IP-filter refresh loop.
**Summary:** The asynchronous refresh path performs hostname resolution through a synchronous socket API. Resolver latency therefore blocks unrelated work on the same event loop.
**Impact:** Signing, publisher scheduling, health checks, blockchain refreshes, and allowlist maintenance can be delayed together.
**Evidence:** OMoC-Node/servers/common/helpers.py:108; servers/oracle/src/ip_filter_loop.py:24.
**Disposition:** Accepted for remediation.

### OI-04 — Peer HTTP inputs and outstanding requests are not resource-bounded
**Affected components:** OMoC-Node HTTP client, signing API, and signature collection.
**Summary:** Outbound responses are fully buffered and decompressed without a size limit, inbound signing has no application-level concurrency or body limits, and unused peer requests are not always explicitly cancelled after quorum.
**Impact:** Peer communication can retain excessive memory, CPU, connections, and asynchronous tasks, reducing publication reliability.
**Evidence:** OMoC-Node/servers/common/helpers.py:37; servers/oracle/src/app.py:111; servers/oracle/src/oracle_coin_pair_loop.py:177; servers/common/run_uvicorn.py:21.
**Disposition:** Accepted for remediation.

### OI-05 — Oversized oracle endpoint strings amplify RPC and decoding work
**Affected components:** OMoC-Decentralized-Oracle registration and InfoGetter; OMoC-Node configuration refresh.
**Summary:** Endpoint strings have no protocol-level length limit and are returned in aggregate selected-oracle queries. Every refresh repeatedly reads, ABI-encodes, transports, decodes, parses, and may log the complete values.
**Impact:** Large stored metadata can exceed RPC limits or cause recurring memory and decoding pressure across the oracle fleet.
**Evidence:** OMoC-Decentralized-Oracle/contracts/libs/IterableOraclesLib.sol:47; contracts/InfoGetter.sol:140.
**Disposition:** Accepted for remediation.

### OI-06 — Price observation timestamps are publisher-controlled and unsigned
**Affected components:** OMoC-Node publication messages and price validation.
**Summary:** The timestamp used to select a historical market observation is not part of the signed price payload. Identical signed price bytes can therefore be evaluated against different observation times by different peers.
**Impact:** Quorum signatures do not prove that signers validated the price against the same market observation.
**Evidence:** OMoC-Node/servers/oracle/src/oracle_publish_message.py:41,68; servers/oracle/src/coin_pair_runner.py:68; servers/oracle/src/price_feeder/price_feeder.py:63.
**Disposition:** Accepted for remediation.

### OI-07 — Submitted signature subsets determine oracle activity accounting
**Affected components:** OMoC-Node signature collection; OMoC-Decentralized-Oracle missed-round accounting.
**Summary:** Collection stops at bare quorum, while the contract credits activity only to signatures included in the publication. Responsive selected oracles omitted from the final subset are indistinguishable from unavailable participants.
**Impact:** Honest participants can accumulate missed rounds and be automatically unsubscribed when missed-signature enforcement is enabled.
**Evidence:** OMoC-Node/servers/oracle/src/oracle_coin_pair_loop.py:177; OMoC-Decentralized-Oracle/contracts/RoundManager.sol:483,505.
**Disposition:** Accepted for remediation.

### OI-08 — Oracle signatures lack deployment and message-type domain separation
**Affected components:** CoinPairPrice, TasksRunner, and OMoC-Node message encoding.
**Summary:** Signed payloads omit chain ID, verifying contract, and an explicit message type, while price and task messages share the same field layout.
**Impact:** Identical identifiers, publishers, signer sets, and nonces across deployments or message classes can make signatures valid outside their intended domain.
**Evidence:** OMoC-Decentralized-Oracle/contracts/CoinPairPrice.sol:121; contracts/TasksRunner.sol:254; OMoC-Node/servers/oracle/src/oracle_publish_message.py:41.
**Disposition:** Accepted for remediation.

### OI-09 — Even-source weighted median combines values from inconsistent orderings
**Affected components:** moc_prices_source weighted median implementation.
**Summary:** The median position is determined in sorted order, but the even-count calculation combines elements using original-list positions. The result depends on source ordering and may not be a median.
**Impact:** Active multi-source pairs can produce incorrect composite prices, including pairs used in RIF-related calculations.
**Evidence:** moc_prices_source/moc_prices_source/weighing.py:198,237; plugins/pairs/computed.py:120; OMoC-Node/servers/oracle/src/price_feeder/moc_price_engines.py:411.
**Disposition:** Accepted and fixed.

### OI-10 — Concurrent price requests share mutable engine and multicall state
**Affected components:** moc_prices_source engines, plugins, EVM state, and HTTP server.
**Summary:** Engine instances, output fields, session data, EVM objects, and multicall tables are shared across concurrent requests without request isolation.
**Impact:** Simultaneous requests can overwrite or consume each other’s intermediate state, returning incorrect prices, incorrect liquidation flags, or failures.
**Evidence:** moc_prices_source/moc_prices_source/engines.py:4; plugins/base.py:412; evm.py:479; server.py:645.
**Disposition:** Accepted for remediation.

### OI-11 — DOC/USD validity is discarded before liquidation calculations
**Affected components:** moc_prices_source on-chain DOC/USD engine and RoC computed pairs.
**Summary:** The on-chain price adapter discards the validity boolean returned by the provider and passes the stored value into liquidation and micro-liquidation calculations.
**Impact:** Expired or invalid data can produce incorrect liquidation signals. Applicability depends on the deployed price-library branch.
**Evidence:** moc_prices_source/moc_prices_source/plugins/engines/onchain/doc_usd_tec.py:12; plugins/pairs/computed_onchain.py:56.
**Disposition:** Accepted for remediation.

### OI-12 — Important price pairs remain valid with only one surviving source
**Affected components:** moc_prices_source pair configuration and weight normalization.
**Summary:** Failed sources are removed before weights are normalized, several pairs require only one successful source, and some RIF-related feeds assign one venue the complete configured weight.
**Impact:** Source redundancy can collapse without invalidating the price, leaving one provider to determine the published value.
**Evidence:** moc_prices_source/moc_prices_source/plugins/pairs/simple.py:23; data/weighing.json:28; __init__.py:81.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.

### OI-13 — The elected publisher bypasses conditional publication eligibility checks
**Affected components:** OMoC-Node publisher authorization and conditional publication.
**Summary:** Peer validation accepts the elected publisher before evaluating price-change, expiry, task-availability, or task-delay conditions.
**Impact:** Signatures can be issued when the ordinary conditional-publication criteria do not require a new publication, weakening the intended gating policy.
**Evidence:** OMoC-Node/servers/oracle/src/oracle_turn.py:24,211,333.
**Disposition:** Accepted for remediation

### OI-14 — Publisher selection uses stake measured after election entropy is known
**Affected components:** OMoC-Node publisher selection and OMoC-Decentralized-Oracle stake queries.
**Summary:** Publisher selection uses current stake values rather than values snapshotted before the election block. Participants can therefore change the selector’s weighted inputs after its block-hash entropy is known.
**Impact:** Publisher-selection probabilities can be influenced through post-entropy stake changes.
**Evidence:** OMoC-Node/servers/oracle/src/select_next.py:57-98; OMoC-Decentralized-Oracle/contracts/InfoGetter.sol:177; contracts/Staking.sol:77.
**Credit:** Immunefi report #91222 by Heisenberg92.
**Disposition:** Rejected — no remediation planned. The risk is accepted at its current impact, and changing the publisher schedule would require a coordinated oracle-node migration.

### OI-15 — Minimum-stake increases do not revalidate existing subscriptions
**Affected components:** OMoC-Decentralized-Oracle subscription and governance configuration.
**Summary:** Minimum stake is enforced on subscription and withdrawal, but changing the configured minimum does not recheck already subscribed oracles.
**Impact:** Existing participants below the new threshold can remain eligible for future rounds indefinitely.
**Evidence:** OMoC-Decentralized-Oracle/contracts/RoundManager.sol:91,127; contracts/change/OracleManagerOracleMinStakeChange.sol:31.
**Disposition:** Accepted for remediation.

### OI-16 — Contract configuration permits more selected oracles than nodes support
**Affected components:** OMoC-Decentralized-Oracle round configuration; OMoC-Node publisher selection.
**Summary:** The contracts do not cap selected-oracle count at the node implementation’s supported maximum of 32.
**Impact:** A valid governance configuration and subscriber set can create rounds that current nodes cannot process, halting publication.
**Evidence:** OMoC-Decentralized-Oracle/contracts/libs/RoundInfoLib.sol:37; contracts/change/CoinPairPriceMaxOraclesPerRoundChange.sol:41; OMoC-Node/servers/oracle/src/select_next.py:57.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.

### OI-17 — Owner and signing addresses are mixed in blockchain-information paths
**Affected components:** OMoC-Node oracle information service and OMoC-Decentralized-Oracle InfoGetter.
**Summary:** Some node paths supply selected signing addresses to methods expecting owner addresses, while InfoGetter requests round points using the signing address.
**Impact:** Nodes can receive incorrect endpoint, stake, owner, round, and points information, particularly when InfoGetter is unavailable.
**Evidence:** OMoC-Node/servers/oracle/src/oracle_coin_pair_service.py:59; OMoC-Decentralized-Oracle/contracts/RoundManager.sol:274; contracts/InfoGetter.sol:185.
**Disposition:** Accepted for remediation.

### OI-18 — Node and price library disagree on the liquidation endpoint name
**Affected components:** OMoC-Node conditional publication and moc_prices_source liquidation output.
**Summary:** The node requests `ISLIQ_FLIP`, while the reviewed price-library branch exposes `ISLIQ_ROC`. Request failures preserve a previous or default false value.
**Impact:** Liquidation-aware forced publication can remain disabled despite a positive liquidation signal. Applicability is branch and configuration dependent.
**Evidence:** OMoC-Node/servers/common/settings.py:69; servers/common/services/conditional_publish.py:167; moc_prices_source/moc_prices_source/plugins/pairs/computed_onchain.py:99.
**Disposition:** Accepted for remediation.

### OI-19 — Running nodes do not apply blockchain-configured address changes
**Affected components:** OMoC-Node configuration refresh and service construction.
**Summary:** Configuration values are refreshed, but manager, supporter, and information services retain the contract addresses supplied at startup.
**Impact:** Governance migrations can leave running nodes connected to different contract generations until every process is restarted.
**Evidence:** OMoC-Node/servers/oracle/src/main_loop.py:38; servers/oracle/src/oracle_configuration.py:277.
**Disposition:** Accepted for remediation.

### OI-20 — Price-library RPC timeouts are configured but not applied
**Affected components:** moc_prices_source custom HTTP provider.
**Summary:** Timeout settings reach the provider but are not passed to the underlying HTTP request.
**Impact:** An unavailable RPC endpoint can occupy service workers indefinitely and prevent timely price generation.
**Evidence:** moc_prices_source/moc_prices_source/evm.py:39; plugins/base.py:809.
**Disposition:** Accepted for remediation.

### OI-21 — The price API is unauthenticated and creates per-request worker pools
**Affected components:** moc_prices_source HTTP service.
**Summary:** The service binds broadly with unrestricted CORS and creates worker resources according to selected engines for every request, without application-level admission controls.
**Impact:** Network-accessible deployments can exhaust threads and outbound API or RPC capacity.
**Evidence:** moc_prices_source/moc_prices_source/server.py:63,645; engines.py:64.
**Disposition:** Accepted for remediation.

### OI-22 — Empty TasksRunner configuration causes task publication failure
**Affected components:** OMoC-Decentralized-Oracle TasksRunner.
**Summary:** Initialization and task removal allow an empty task set, while task execution performs a modulo operation using the task count.
**Impact:** All otherwise valid task publications revert until governance restores at least one task.
**Evidence:** OMoC-Decentralized-Oracle/contracts/TasksRunner.sol:83,128,312.
**Disposition:** Rejected — no remediation planned. An empty TasksRunner has no intended purpose.

### OI-23 — IP allowlist cleanup mutates its dictionary during iteration
**Affected components:** OMoC-Node IP-filter refresh.
**Summary:** Removing stale addresses while iterating the same dictionary raises a runtime exception and interrupts the refresh cycle.
**Impact:** Stale addresses can remain authorized longer than intended and subsequent entries may not be processed during the failed cycle.
**Evidence:** OMoC-Node/servers/oracle/src/ip_filter_loop.py:49.
**Disposition:** Accepted for remediation.

### OI-24 — Remote weight validation replaces configuration with a boolean
**Affected components:** moc_prices_source remote weighing configuration.
**Summary:** Remote JSON validation returns a boolean that is assigned in place of the downloaded weight mapping. The download request also lacks a timeout.
**Impact:** Remote-weight mode can fail, apply no authoritative weights, or remain blocked waiting for the remote service.
**Evidence:** moc_prices_source/moc_prices_source/weighing.py:78,103.
**Disposition:** Accepted for remediation.

### OI-25 — Publication signatures have no independent expiration
**Affected components:** OMoC-Node signing flow and OMoC-Decentralized-Oracle CoinPairPrice.
**Summary:** Signature validity is tied to publication fields and nonce progression rather than a signer-enforced block or time deadline. Conditional publication can leave a valid signature set unused for an extended interval.
**Impact:** Signatures may remain consumable after the market context in which they were issued has changed, until another publication advances the relevant nonce.
**Evidence:** OMoC-Decentralized-Oracle/contracts/CoinPairPrice.sol:121; OMoC-Node/servers/oracle/src/oracle_publish_message.py:41; servers/oracle/src/oracle_turn.py:211.
**Disposition:** Accepted for remediation.

### OI-26 — Emergency publication demand is not latched after detection
**Affected components:** chain_monitor emergency feeder.
**Summary:** The feeder reevaluates the current age and price-difference conditions on every block and resets its delay counter when they no longer hold. It does not record a durable publication obligation after first detection.
**Impact:** A transient condition can be observed without producing the delayed emergency publication that the initial observation appeared to require.
**Evidence:** chain_monitor/chain_monitor/omoc_feeder/daemon.py:347-394,408-436.
**Disposition:** Accepted for remediation.

### OI-27 — Medianizer validity is cached rather than recomputed on reads
**Affected components:** Amphiraos Medianizer and downstream oracle consumers.
**Summary:** The validity boolean returned by reads reflects the last update operation rather than a live freshness and quorum evaluation.
**Impact:** Previously accepted values can continue to appear valid after their underlying freshness or quorum conditions change.
**Evidence:** Amphiraos-Oracle Medianizer `poke` and `peek` validity handling.
**Disposition:** Rejected — no remediation planned.

### OI-28 — Legacy Medianizer values do not expire automatically
**Affected components:** Legacy Amphiraos-Oracle deployment.
**Summary:** The reviewed implementation does not attach an expiring timestamp to updates, and a successful validity flag remains set until another state-changing action modifies it.
**Impact:** Consumers can continue receiving an old value as valid when feeders stop updating. The previously reported deployment was assessed as inactive.
**Evidence:** Amphiraos-Oracle Medianizer and DSValue update paths referenced by the prior Immunefi report.
**Disposition:** Rejected — no remediation planned.

### OI-29 — Medianizer invalidation does not prevent later reactivation
**Affected components:** Legacy Amphiraos-Oracle Medianizer.
**Summary:** The invalidation function clears validity but does not disable subsequent permissionless recomputation, so it is not a persistent circuit breaker.
**Impact:** The contract cannot enforce a lasting oracle freeze at the Medianizer layer. The reviewed deployment was assessed as inactive and its sources centrally controlled.
**Evidence:** Amphiraos-Oracle Medianizer `void` and `poke` paths referenced by the prior Immunefi report.
**Disposition:** Rejected — no remediation planned.

### OI-30 — Oracle signature recovery accepts non-canonical high-s signatures
**Affected components:** OMoC-Decentralized-Oracle RoundManager signature recovery; OMoC-SC-Shared oracle integration.
**Summary:** The shared recovery path calls `ecrecover` directly and validates `v`, but does not require the canonical low-s representation enforced by standard ECDSA helpers.
**Impact:** The same signer and payload can have more than one valid signature representation. This does not create additional quorum weight because execution deduplicates signatures by recovered address, so the issue is limited to signature canonicalization and defense in depth.
**Evidence:** OMoC-Decentralized-Oracle/contracts/RoundManager.sol:427-442,483-495; OMoC-SC-Shared bundled OMoC-Node signature handling in servers/common/crypto.py:22-47.
**Disposition:** Rejected — no remediation planned. Duplicate signer weight is already prevented by recovered-address ordering.

### OI-31 — Signature recovery does not explicitly reject the zero address
**Affected components:** OMoC-Decentralized-Oracle RoundManager signature recovery and execution validation.
**Summary:** A failed `ecrecover` call can return `address(0)` and the recovery helper returns that value without an explicit check. The surrounding validation nevertheless rejects it through the strict ascending-address requirement.
**Impact:** Invalid signatures can fail with the generic ordering error instead of a dedicated recovery error; they are not counted toward quorum. Non-reverting recovery is also required by the planned compatibility flow that first attempts the expiring-price format and then retries the legacy format.
**Evidence:** OMoC-Decentralized-Oracle/contracts/RoundManager.sol:427-442,483-495.
**Disposition:** Rejected — no remediation planned. The current behavior supports signature-format fallback and does not admit a zero-address signer.

### OI-32 — Signature parameter validation relies on custom recovery logic
**Affected components:** OMoC-Decentralized-Oracle RoundManager, CoinPairPrice, and TasksRunner; OMoC-SC-Shared oracle integration.
**Summary:** Signature recovery uses a project-specific helper that normalizes `v` from 0/1 to 27/28 and calls `ecrecover` directly. Canonical-s checks, recovery errors, and future format fallback must therefore be maintained locally rather than through a standard ECDSA utility.
**Impact:** Cryptographic validation behavior is easier to change inconsistently as new signed message formats and consumers are introduced. No independent quorum bypass is currently demonstrated.
**Evidence:** OMoC-Decentralized-Oracle/contracts/RoundManager.sol:427-442; contracts/CoinPairPrice.sol:103-139; contracts/TasksRunner.sol:237-270; OMoC-SC-Shared bundled OMoC-Node servers/common/crypto.py:11-60.
**Disposition:** Accepted for remediation. Preserve support for `v` values 0/1 and use non-reverting recovery where legacy-format fallback is required.

### OI-33 — Oracle messages are not bound through EIP-712 domain separation
**Affected components:** OMoC-Decentralized-Oracle CoinPairPrice and TasksRunner; OMoC-SC-Shared bundled OMoC-Node message construction.
**Summary:** Price and task signatures use legacy EIP-191 messages that omit the chain ID and verifying contract address. They are therefore not cryptographically bound to one deployment, and the two message classes also use the same field layout.
**Impact:** Matching deployment state and payload fields can permit cross-chain, cross-contract, or cross-message-class replay. This is the concrete EIP-712 remediation track for the replay scope already recorded in OI-08, not a separate replay primitive.
**Evidence:** OMoC-Decentralized-Oracle/contracts/CoinPairPrice.sol:117-138; contracts/TasksRunner.sol:251-270; OMoC-SC-Shared bundled OMoC-Node servers/oracle/src/oracle_publish_message.py:41-63,105-126.
**Disposition:** Accepted for remediation. Migrate price and task payloads to distinct EIP-712 typed-data structures bound to chain ID and verifying contract.

### OI-34 — Oracle message hashing uses manually constructed EIP-191 length prefixes
**Affected components:** OMoC-Decentralized-Oracle CoinPairPrice and TasksRunner; OMoC-SC-Shared bundled OMoC-Node message construction.
**Summary:** Both signed payloads manually embed `"\x19Ethereum Signed Message:\n148"`, coupling signature validity to a hand-maintained byte-length literal and duplicated packed field layout.
**Impact:** A payload change that is not reflected consistently in the prefix and node encoder can invalidate honest signatures or make nodes and contracts hash different messages. The current 148-byte layouts agree, so no present execution failure is demonstrated.
**Evidence:** OMoC-Decentralized-Oracle/contracts/CoinPairPrice.sol:117-138; contracts/TasksRunner.sol:251-270; OMoC-SC-Shared bundled OMoC-Node servers/oracle/src/oracle_publish_message.py:41-63,105-126.
**Disposition:** Accepted for remediation as part of the EIP-712 migration in OI-33.

### OI-35 — TaskTriggerOrder gas griefing

**Affected components:** OMoC-Decentralized-Oracle TaskTriggerOrder reverse-auction task.
**Summary:** The task catches failures from its external reverse-auction call and completes its failure-handling path. A caller can select a gas limit that causes the external call to fail while retaining enough gas for that path.
**Impact:** Griefing.
**Evidence:** OMoC-Decentralized-Oracle/contracts/tasks/mocFlow/reverseAuction/TaskTriggerOrder.sol:43-50.
**Disposition:** Rejected — no remediation planned.

### OI-36 — TasksRunner gas griefing

**Affected components:** OMoC-Decentralized-Oracle TasksRunner.
**Summary:** The task-execution loop catches failures from registered task calls and continues execution. A task-publication caller can select a gas limit that causes a selected task call to fail while retaining enough gas for the catch path.
**Impact:** Griefing.
**Evidence:** OMoC-Decentralized-Oracle/contracts/TasksRunner.sol:312-345.
**Disposition:** Rejected — no remediation planned.

### OI-37 — OMoC-Node publisher selector reuses correlated randomness
**Affected components:** OMoC-Node publisher selection; OMoC-Decentralized-Oracle publication points and reward distribution.
**Summary:** The publisher selector uses the same SHA-256 digest both to shuffle the stake-sorted oracle roster and to choose a coordinate within its cumulative stake. These correlated decisions make first-choice probability depend on roster rank as well as stake, even when the roster and stake snapshot remain fixed.
**Impact:** Publisher-selection probabilities, publication points, and expected reward shares can deviate from stake-proportional selection according to roster rank.
**Evidence:** OMoC-Node/servers/oracle/src/select_next.py:75-93; OMoC-Decentralized-Oracle/contracts/libs/SubscribedOraclesLib.sol:63-93; contracts/RoundManager.sol:389-414.
**Credit:** Immunefi report #91222 by Heisenberg92.
**Disposition:** Rejected — no remediation planned. The demonstrated bias is accepted at its current impact, and changing the publisher schedule would require a coordinated oracle-node migration.

## Stablecoin Protocol

### SP-01 — Deferred queues permit execution against prices older than the queued operation
**Affected components:** main-sc-protocol-v2 MocQueue; RoC multicollateral deployment.
**Summary:** Once the configured maximum waiting-block threshold is reached, queue eligibility no longer requires a price publication newer than the operation. Checked-in RoC values are one to three blocks.
**Impact:** Deferred operations can execute using an unchanged protocol price even though the operation was submitted after that price was published.
**Evidence:** main-sc-protocol-v2/contracts/queue/MocQueue.sol:861; roc-sc-protocol-v2/contracts/changers/addTasksRunnerChanger/AddTasksRunnerChanger.sol:347; ignition/deployments/chain-30/journal.jsonl:133,171.
**Disposition:** Accepted for remediation.

### SP-02 — Liquidation swaps expose protocol value through broad AMM execution bounds
**Affected components:** main-sc-protocol-v2 MocRebalancer and MocSwapperV3; RoC bucket configuration.
**Summary:** Public liquidation and micro-liquidation functions execute exact-input AMM trades whose minimum outputs are derived from configurable oracle-relative penalties, without a pool-price limit.
**Impact:** The difference between oracle value and the permitted minimum output can be lost from protocol collateral during rebalancing, subject to configured trade caps.
**Evidence:** main-sc-protocol-v2/contracts/multiCollateral/MocRebalancer.sol:373,432,455,667,728; contracts/multiCollateral/swapper/MocSwapperV3.sol:159; roc-sc-protocol-v2/ignition/deployments/chain-30/journal.jsonl:239.
**Disposition:** Rejected — no remediation planned. Accepted risk.

### SP-03 — Zero-penalty liquidation routes can be non-executable
**Affected components:** main-sc-protocol-v2 MocRebalancer; RoC DOC bucket configuration.
**Summary:** A zero penalty requires approximately the complete oracle-equivalent output from an AMM trade, without accounting for normal pool fees and price impact.
**Impact:** Required liquidation or micro-liquidation operations can repeatedly revert until market conditions or governance parameters change.
**Evidence:** main-sc-protocol-v2/contracts/multiCollateral/MocRebalancer.sol:373,432; roc-sc-protocol-v2/ignition/deployments/chain-30/journal.jsonl:239.
**Disposition:** Accepted for remediation.

### SP-04 — Reverse auctions discard the oracle validity flag
**Affected components:** main-sc-protocol-v2 MocReverseAuction.
**Summary:** Minimum output is calculated from returned price bytes without checking the accompanying validity boolean.
**Impact:** Auctioned protocol funds can be converted using a retained stale value or an invalid zero value, although configured trade caps and the project’s reasonable-price policy limit exposure.
**Evidence:** main-sc-protocol-v2/contracts/auxiliary/MocReverseAuction.sol:235; contracts/interfaces/IPriceProvider.sol.
**Disposition:** Rejected — no remediation planned.

### SP-05 — Reverse-auction demand is not a conditional publication trigger
**Affected components:** RoC reverse auctions, conditional publication service, and emergency feeder.
**Summary:** Conditional publication criteria do not include accumulated reverse-auction demand, while the auction may continue treating an older retained oracle value as usable.
**Impact:** Auction execution can rely on prices substantially older than the normal publication cadence during stable periods.
**Evidence:** main-sc-protocol-v2/contracts/auxiliary/MocReverseAuction.sol:235; OMoC-Node/servers/common/services/conditional_publish.py; chain_monitor/chain_monitor/omoc_feeder/daemon.py:347-419.
**Disposition:** Rejected — no remediation planned.

### SP-06 — Reverse-auction trigger thresholds are advisory only
**Affected components:** main-sc-protocol-v2 MocReverseAuction.
**Summary:** The readiness view checks the configured accumulated-balance threshold, but the state-changing trigger function does not enforce the same condition.
**Impact:** Auctions can execute before reaching their configured batch threshold, changing trade frequency, cost efficiency, and price exposure.
**Evidence:** main-sc-protocol-v2/contracts/auxiliary/MocReverseAuction.sol:113,136.
**Disposition:** Rejected — no remediation planned. Expected behavior.

### SP-07 — Zero-value deferred operations consume queue throughput
**Affected components:** main-sc-protocol-v2 operation entrypoints and MocQueue.
**Summary:** Queue entrypoints accept zero token quantities, while execution rejects them later. Failed entries are removed but still occupy batch-processing capacity.
**Impact:** Paid non-executable entries can reduce automation throughput and delay legitimate operations, particularly when execution fees are low.
**Evidence:** main-sc-protocol-v2/contracts/core/MocOperations.sol:108; contracts/core/MocCore.sol:162; contracts/queue/MocQueue.sol:909; roc-sc-protocol-v2/ignition/deployments/chain-30/journal.jsonl:135.
**Disposition:** Accepted for remediation.

### SP-08 — Deferred operations provide outcome-contingent execution
**Affected components:** main-sc-protocol-v2 MocQueue and operation parameter structures.
**Summary:** Queued operations include user price limits but no execution deadline. They execute against future oracle and protocol state, while failed economic conditions return principal and consume only the execution fee.
**Impact:** Users receive asymmetric price optionality, which becomes more consequential when stale-price queue execution is also possible.
**Evidence:** main-sc-protocol-v2/contracts/core/MocCore.sol:222; contracts/core/MocCommons.sol:130; contracts/queue/MocQueue.sol:621.
**Disposition:** Rejected — no remediation planned. By design; the optionality is short-lived.

### SP-09 — An empty registered target bucket can block liquidation
**Affected components:** main-sc-protocol-v2 MocRebalancer.
**Summary:** Multicollateral target selection excludes economically empty buckets but the full-liquidation path lacks a zero-target fallback when more than one bucket is registered.
**Impact:** A bucket requiring liquidation can remain unliquidatable while the only alternative registered bucket is empty.
**Evidence:** main-sc-protocol-v2/contracts/multiCollateral/MocRebalancer.sol:193,735.
**Disposition:** Accepted for remediation.

### SP-10 — Execution fees are repriced at execution from pooled funds
**Affected components:** main-sc-protocol-v2 MocQueue fee accounting and MocMultiCollateralGuard.
**Summary:** Users prepay using the enqueue-block base fee, but executor and price-update payments are recalculated using the execution-block base fee and paid from aggregate balances.
**Impact:** Base-fee changes can shift costs between queued users and reduce funds available for price-update compensation.
**Evidence:** main-sc-protocol-v2/contracts/queue/MocQueueExecFees.sol:114; contracts/queue/MocQueue.sol:923; contracts/multiCollateral/MocMultiCollateralGuard.sol:662.
**Disposition:** Rejected — no remediation planned. Accepted risk.

### SP-11 — V3 lending swapper reports quoted rather than actually minted TP
**Affected components:** lending-and-borrowing-sc MocSwapperCoreV3 and lending accounting.
**Summary:** Swapper methods discard the actual minted amount returned by the MoC operation and report a precomputed output. Downstream debt and liquidity accounting trusts that reported value without checking the received balance.
**Impact:** Lending state can recognize TP that was not received when actual minting differs from the quote.
**Evidence:** lending-and-borrowing-sc/contracts/interfaces/IMocV3.sol; contracts/swappers/MocSwapperCoreV3.sol; contracts/MocLendingManager.sol; contracts/lib/MocLendingPool.sol.
**Disposition:** Accepted for remediation.

### SP-12 — Lending pool initialization omits critical parameter validation
**Affected components:** lending-and-borrowing-sc pool initialization and borrowing.
**Summary:** Initialization stores token, injector, coverage, fee, utilization, and brake parameters without the bounds previously enforced by the pool.
**Impact:** Invalid governed initialization can permit borrowing without meaningful collateral checks or make interest and pool operations revert after deployment.
**Evidence:** lending-and-borrowing-sc/contracts/MocLendingManager.sol; contracts/lib/MocLendingPool.sol.
**Disposition:** Accepted for remediation.

### SP-13 — TP-to-TP swaps do not update flux-capacitor limits
**Affected components:** main-sc-protocol-v2 TP swap and flux-capacitor accounting.
**Summary:** The TP-to-TP path burns the source and mints the target without applying the accumulator updates used by comparable TP mint and redeem operations.
**Impact:** TP exposure can move without consuming the configured operation-rate limits. The project previously classified this behavior as outside the intended flux-capacitor use case.
**Evidence:** main-sc-protocol-v2/contracts/core/MocCoreExpansion.sol; contracts/core/MocCore.sol; contracts/core/MocCommons.sol; contracts/collateral/rc20/MocCARC20.sol.
**Disposition:** Rejected — no remediation planned.

### SP-14 — RC20 collateral balance refresh is unsafe during callback-capable transfers
**Affected components:** main-sc-protocol-v2 MocCARC20 collateral accounting.
**Summary:** The public balance-refresh function credits the difference between actual token balance and accounted collateral. It can observe temporary transfer balances before the enclosing operation completes when the configured token permits callbacks.
**Impact:** Accounted collateral can exceed actual reserves, distorting coverage and token pricing. Standard non-callback collateral tokens do not expose this condition.
**Evidence:** main-sc-protocol-v2/contracts/collateral/rc20/MocCARC20.sol:103; contracts/core/MocCore.sol; contracts/core/MocBaseBucket.sol.
**Disposition:** Accepted for remediation.

### SP-15 — Lending swap slippage parameters have no upper bound
**Affected components:** lending-and-borrowing-sc manager, TP injector, and swap calculations.
**Summary:** Initialization and governance setters accept a maximum-slippage value above the precision unit, while swap calculations subtract it from that unit using checked arithmetic.
**Impact:** An invalid parameter can make repayment, liquidation, and TP-injection flows revert until governance corrects it.
**Evidence:** lending-and-borrowing-sc/contracts/MocLendingManager.sol; contracts/TPInjector.sol; contracts/lib/Coin.sol.
**Disposition:** Accepted for remediation.

### SP-16 — Zero-address allocation shares remain available for redistribution
**Affected components:** main-sc-protocol-v2 MocTokenAllocator and ratio sources.
**Summary:** A zero recipient causes its transfer to be skipped, leaving the intended share in the allocator. Later splits recalculate all shares from the remaining balance, and ratio sources do not consistently reject zero recipients.
**Impact:** Funds intended for an unset recipient can be progressively redistributed to other configured recipients.
**Evidence:** main-sc-protocol-v2/contracts/auxiliary/MocTokenAllocator.sol; contracts/auxiliary/ratio-sources/TokenRatioSource.sol; contracts/auxiliary/ratio-sources/TVLRatioSource.sol.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.

### SP-17 — Settlement cannot skip flux-capacitor reverts
**Affected components:** Legacy RDOC settlement and StableToken redemption.
**Summary:** Settlement’s partial-execution engine handles a false result but not a revert from the flux-capacitor limit in the redemption path.
**Impact:** One over-limit settlement entry can prevent progress past the same queue position. The affected settlement and BTCX functionality is deprecated and inactive.
**Evidence:** Legacy settlement and StableToken flux-capacitor paths referenced by the prior Immunefi report.
**Disposition:** Rejected — no remediation planned.

### SP-18 — Failed queued-operation refunds rely on dormant locked-fund accounting
**Affected components:** main-sc-protocol-v2 MocQueue refund handling.
**Summary:** When an AC refund transfer fails, the queue records the amount in sender and total locked-fund variables, removes the operation, and exposes no current user self-service recovery path.
**Impact:** Refunded collateral can remain unavailable pending governance intervention. The project treats best-effort refunds and governance recovery as intended behavior.
**Evidence:** main-sc-protocol-v2/contracts/queue/MocQueue.sol:512 and locked-fund accounting paths.
**Disposition:** Rejected — no remediation planned.

### SP-19 — Post-liquidation payments can reduce the frozen TP redemption pool
**Affected components:** main-sc-protocol-v2 settlement, TC-holder interest, and liquidation redemption.
**Summary:** Settlement and TC-holder interest functions remain callable after liquidation and can move AC or create claims after TP liquidation prices have been frozen.
**Impact:** The collateral remaining for TP liquidation redemptions can differ from the frozen claim basis. The project treats fee and interest priority as a governance policy choice.
**Evidence:** main-sc-protocol-v2 MocCore `execSettlement`, `tcHoldersInterestPayment`, and liquidation redemption paths.
**Disposition:** Rejected — no remediation planned.

### SP-20 — Cross-bucket freshness aggregation can wake unrelated queues
**Affected components:** main-sc-protocol-v2 MocMultiCollateralGuard and per-bucket queues.
**Summary:** A guard-level aggregate publication marker can signal queue execution based on a sibling bucket’s update, although each operation is associated with its own price feed.
**Impact:** Misconfiguration can increase failed or cancelled operations. The project states that individual execution rechecks freshness and that the current RoC setting is intentional.
**Evidence:** main-sc-protocol-v2 multicollateral publication-block aggregation and queue freshness checks referenced by the prior Immunefi report.
**Disposition:** Rejected — no remediation planned.

### SP-21 — Liquidation redemptions can consume collateral reserved for pending queue entries
**Affected components:** main-sc-protocol-v2 liquidation redemption and queue cleanup.
**Summary:** Liquidation claims use actual bucket balance while pending queued AC is tracked separately, allowing liquidation redemption and later queue refund accounting to compete for the same balance.
**Impact:** Pending collateral can become unavailable during an exceptional liquidation ordering. Automatic liquidation is currently disabled and the project treats this as an accepted liquidation-policy edge case.
**Evidence:** main-sc-protocol-v2 bucket liquidation, TP redemption, MocQueue cleanup, and locked-fund paths referenced by the prior Immunefi report.
**Disposition:** Rejected — no remediation planned.

### SP-22 — Reserve-token vendor markup bypasses the staking cap calculation
**Affected components:** Legacy main-RBTC-contract MoCVendors and fee routing.
**Summary:** The eligibility check uses the direct MoC amount before converting and adding reserve-token markup, so reserve-token payments contribute zero to the pre-payment cap check.
**Impact:** Vendor markup can be paid beyond configured staking capacity instead of following the fallback fee-routing policy. Vendor functionality is privately controlled and planned for removal.
**Evidence:** main-RBTC-contract/contracts/MoCVendors.sol; contracts/MoC.sol; contracts/MoCExchange.sol.
**Disposition:** Rejected — no remediation planned.

### SP-23 — Liquidation state transitions can accept RBTC without executing payable operation accounting
**Affected components:** Legacy main-RBTC-contract payable fallback and mint paths.
**Summary:** The state-transition modifier can enter liquidation and skip the function body after RBTC has been received, omitting minting and internal collateral accounting.
**Impact:** Incoming RBTC can remain unaccounted and no requested token output is produced. Liquidation is disabled and requires governance action in the current system.
**Evidence:** main-RBTC-contract/contracts/MoC.sol transition-state modifier, fallback, and payable mint paths.
**Disposition:** Rejected — no remediation planned.

### SP-24 — Legacy RBTC fee transfers were reported incompatible with the commissions recipient
**Affected components:** Legacy main-RBTC-contract RBTC commission forwarding.
**Summary:** A prior report claimed the production recipient could not accept the fixed-gas transfer used for RBTC commissions, causing normal fee-paying operations to revert.
**Impact:** The project rejected the premise after confirming those production operations continue to execute and assessed the fork result as nonrepresentative.
**Evidence:** main-RBTC-contract/contracts/MoC.sol commission-transfer paths and the production recipient cited in the prior Immunefi report.
**Disposition:** Rejected — no remediation planned.

### SP-25 — Token migration can be initiated by a third party after approval
**Affected components:** roc-sc-protocol-v2 TokenMigrator.
**Summary:** Migration of an approved holder’s full legacy balance does not require the transaction sender to be that holder. New tokens are still credited to the same holder.
**Impact:** Holders do not exclusively control migration timing after granting allowance. The project considers permissionless execution an intentional relayer and automation feature.
**Evidence:** roc-sc-protocol-v2/contracts/TokenMigrator.sol:78.
**Disposition:** Rejected — no remediation planned.

### SP-26 — MOC commission pricing uses a manipulable 120-second TWAP
**Affected components:** Legacy main-RBTC-contract MoCExchange and MoCState.
**Summary:** MOC-denominated commissions are calculated using the MOC price from a 120-second Uniswap V3 TWAP. Because that price is used as the divisor, temporary market movements can increase or decrease the MOC amount charged for protocol fees.
**Impact:** Users may pay more or fewer MOC tokens than expected while the TWAP is displaced. This does not transfer the excess fee to the manipulator; it is collected by the protocol. Users can avoid MOC-denominated pricing by paying fees in RBTC.
**Evidence:** `MoCExchange.calculateCommissionsWithPrices` and the `MoCState.getMoCPriceProvider()` 120-second MOC TWAP.
**Disposition:** Rejected — no remediation planned. The 120-second TWAP is considered a fair market price that reflects actual MOC trading activity, and users retain the option to pay fees in RBTC.

### SP-27 — Reverse auctions may use cached prices after oracle invalidation
**Affected components:** Legacy main-RBTC-contract MocReverseAuction.
**Summary:** `MocReverseAuction._getAmountOutMin` uses the cached price from `IPriceProvider.peek()` even when the provider marks it invalid.
**Impact:** A stale price may weaken slippage protection and result in worse execution for accumulated protocol fee funds.
**Evidence:** `MocReverseAuction._getAmountOutMin` ignores the validity flag returned by `IPriceProvider.peek()`.
**Disposition:** Rejected — reverse auctions prioritize continued market clearing for limited fee balances. The cached price and configured slippage provide an adequate execution reference; no remediation planned.

### SP-28 — Reverse-auction TWAP and execution venue share the same pool
**Affected components:** Legacy main-RBTC-contract MocReverseAuction and MocSwapperV3MultiHop.
**Summary:** The reverse auction derives `amountOutMin` from a 120-second MOC TWAP that includes the same WRBTC/MOC pool used by the final swap. Pool price movements can therefore affect both the minimum-output calculation and the swap execution price.
**Impact:** A displaced pool price may cause an auction conversion to deliver fewer MOC tokens to the RewardsBuffer than an unmanipulated conversion.
**Evidence:** `MocReverseAuction._getAmountOutMin`, `triggerOrders`, and the RIF → USDRIF → WRBTC → MOC route using the WRBTC/MOC pool.
**Disposition:** Rejected — the 120-second TWAP and shared execution venue are intentional design choices for reverse-auction fee conversion. No remediation planned.

### SP-29 — stable-protocol-core-v2 MocQueue gas griefing

**Affected components:** stable-protocol-core-v2 MocQueue deferred-operation executor.
**Summary:** Deferred-operation execution catches failures from external operation calls and completes its failure-handling path. A public executor can select a gas limit that causes an operation call to fail while retaining enough gas for the catch path.
**Impact:** Griefing.
**Evidence:** stable-protocol-core-v2/contracts/queue/MocQueue.sol:475-805,827-858.
**Disposition:** Rejected — no remediation planned.

### SP-30 — MocLendingQueue gas griefing

**Affected components:** lending-and-borrowing-sc MocLendingQueue and MocLendingManager.
**Summary:** Queued lending-operation execution catches failures from its self-call and completes its failure-handling path. A public executor can select a gas limit that causes the self-call to fail while retaining enough gas for the catch path.
**Impact:** Griefing.
**Evidence:** lending-and-borrowing-sc/contracts/MocLendingQueue.sol:226-261; contracts/MocLendingManager.sol:980-984.
**Disposition:** Rejected — no remediation planned.

### SP-31 — main-sc-protocol-v2 MocQueue gas griefing

**Affected components:** main-sc-protocol-v2 MocQueue and MocMultiCollateralGuard execution path.
**Summary:** Deferred-operation execution catches failures from external operation calls and completes its failure-handling path. An executor using the guard path can select a gas limit that causes an operation call to fail while retaining enough gas for the catch path.
**Impact:** Griefing.
**Evidence:** main-sc-protocol-v2/contracts/queue/MocQueue.sol:542-793,821-927.
**Disposition:** Rejected — no remediation planned.

### SP-32 — TPInjector swap handling gas griefing

**Affected components:** lending-and-borrowing-sc TPInjector and Coin swap helpers.
**Summary:** Swap helpers catch external swap failures and return through their non-reverting path when used by TP injection. A caller can select a gas limit that causes a swap call to fail while retaining enough gas for that path.
**Impact:** Griefing.
**Evidence:** lending-and-borrowing-sc/contracts/lib/Coin.sol:88-205,220-262; contracts/TPInjector.sol:190-200.
**Disposition:** Rejected — no remediation planned.

### SP-33 — Reverse-auction swaps may be unavailable because of path or threshold configuration
**Affected components:** `MocReverseAuction`, `MocSwapperV3MultiHop`, and their deployed path and maximum-amount configuration.
**Summary:** Reverse-auction execution depends on operationally configured swap paths and amount thresholds. A required pair may have no configured path, a multihop path may reference a route that is no longer available, or the configured threshold may prevent useful execution. These configurations are actively monitored and may be updated as market routes change.
**Impact:** An affected reverse auction may be unable to convert its accumulated balance until its swapper configuration is corrected. Failed swaps revert atomically and leave the auction funds pending.
**Evidence:** Reverse auction `0x38ed1F563E75D88cf4Db5Db07972B143b3c8Ebfb` uses swapper `0x24122d7FF0EF57C18e5C333E2c7bD863e4F23c73`, where the DOC/MOC path is not configured. Applicability depends on whether the auction receives DOC through bucket fees.
**Disposition:** Rejected — no remediation planned. Swapper paths and thresholds are operational configuration monitored by the project, and reports limited to missing, unavailable, or unsuitable swap routes are treated as known configuration issues.

### SP-34 — Combined DOC redemption loses TC proportion through scale truncation
**Affected components:** `main-sc-protocol-v2@3.0.9` `MocCoreExpansion.redeemTCandTP`; deployed DOC bucket `0x697535055Aa7AfD2C280523C7B062b1F05284661`.
**Summary:** The combined TC+TP redemption proportion mixes protocol-scale values with a lower-scale intermediate before taking `Math.min`. For DOC, the configured TP target is `1.000000000000000001`, so the `(ctargemaTP - 1)` term is one wei and that arm truncates to 1 (or 0 below the combined target). `Math.min` selects the truncated arm instead of the intended proportion.
**Impact:** Combined redemptions can burn only dust TC and effectively execute as TP-only redemptions; once combined coverage is below target, the zero proportion makes every DOC combined redemption revert. Failed queued attempts still consume the prepaid execution fee.
**Evidence:** DOC bucket TP target configuration and `MocCoreExpansion.redeemTCandTP` proportion calculation in deployed `main-sc-protocol-v2@3.0.9`.
**Disposition:** Unfixed — remediation may be considered later.

## Governance

### GOV-01 — Failed execution permanently consumes an accepted proposal
**Affected components:** MoC-Gobernanza VotingMachine; stable-protocol-interface-v3 governance UI.
**Summary:** The permissionless accepted step leaves the Accepted state before calling the approved changer, catches every execution failure, and provides no retry path.
**Impact:** A correctly approved proposal can be finalized as unsuccessful without applying its change, while the outer transaction still succeeds.
**Evidence:** MoC-Gobernanza/contracts/VotingMachine.sol:265; stable-protocol-interface-v3/src/components/Voting/Vote.tsx:268; src/backend/omoc/voting.ts:168.
**Disposition:** Rejected — no remediation planned. Accepted risk.

### GOV-02 — Pre-vote voting power can become unlocked before the final vote ends
**Affected components:** MoC-Gobernanza VotingMachine; OMoC-Decentralized-Oracle staking and Supporters.
**Summary:** Pre-vote stake is locked for one voting period beginning at pre-vote time, while final voting starts a new full voting period and carries the pre-votes forward without extending their locks.
**Impact:** Counted pre-votes can cease to be backed by locked stake; after withdrawal, the underlying MOC can later receive new voting weight under another address. The original address cannot directly recount the same recorded balance.
**Evidence:** MoC-Gobernanza/contracts/VotingMachine.sol:109,201; contracts/libs/VotingDataLib.sol:37; OMoC-Decentralized-Oracle/contracts/Supporters.sol:386.
**Disposition:** Accepted for remediation.

### GOV-03 — Veto subsystem failures are treated as zero veto participation
**Affected components:** MoC-Gobernanza VotingMachine and RegistryValues.
**Summary:** Failure to locate or query the VetoMachine is caught and converted into zero veto power before proposal acceptance is evaluated.
**Impact:** A proposal can be accepted without considering stored veto participation while the veto subsystem is unavailable or misconfigured.
**Evidence:** MoC-Gobernanza/contracts/libs/RegistryValues.sol:48; contracts/VotingMachine.sol:219,358.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.

### GOV-04 — Governance commits to a changer address rather than immutable contents
**Affected components:** MoC-Gobernanza, MoC-Periphery Governor, and mutable changer implementations.
**Summary:** Voting records only a changer address and does not commit to its bytecode, implementation, parameters, external dependencies, or eventual execution payload.
**Impact:** The privileged action executed after approval can differ from the implementation or state reviewed during voting. The project considers flexible changer behavior an explicit governance policy.
**Evidence:** MoC-Periphery/contracts/moc-governance/Governance/ChangeContract.sol:5; main-sc-protocol-v2/contracts/governance/changerTemplates/EditBucketChangerTemplate.sol:51; EditPeggedTokenChangerTemplate.sol:50.
**Disposition:** Rejected — no remediation planned.

### GOV-05 — Reusing a changer address overwrites historical proposal results
**Affected components:** MoC-Gobernanza VotingMachine storage and proposal getters.
**Summary:** Proposal addresses are appended to an historical list, but vote and execution results are stored in a mapping keyed only by changer address and overwritten on reuse.
**Impact:** Earlier list entries can display a later round’s votes, result, veto amount, and execution status instead of their original history.
**Evidence:** MoC-Gobernanza/contracts/VotingMachineStorage.sol:56-72; contracts/VotingMachine.sol:246-254.
**Disposition:** Rejected — no remediation planned. Accepted behavior; changer reuse is considered unlikely.

### GOV-06 — Veto deposits are not separated by voting round
**Affected components:** MoC-Gobernanza VetoMachine storage and withdrawal.
**Summary:** Veto percentages are keyed by proposal and round, but deposited token balances are keyed only by voter, proposal, and token.
**Impact:** Reusing a proposal address can keep earlier-round collateral unavailable during a later round even though it contributes no veto power to that round.
**Evidence:** MoC-Gobernanza/contracts/VetoMachineStorage.sol:20-24; contracts/VetoMachine.sol:129-150.
**Disposition:** Accepted for remediation.

### GOV-07 — Exact veto threshold is treated as insufficient
**Affected components:** MoC-Gobernanza VotingDataLib.
**Summary:** Veto comparisons use a strict greater-than boundary, while quorum and acceptance treat equality as sufficient.
**Impact:** Participation exactly equal to the configured minimum veto percentage does not veto the proposal, producing inconsistent minimum-threshold semantics.
**Evidence:** MoC-Gobernanza/contracts/libs/VotingDataLib.sol:63-73,78-100.
**Disposition:** Accepted for remediation.

### GOV-08 — Votes remain accepted after the recorded voting expiration
**Affected components:** MoC-Gobernanza VotingMachine.
**Summary:** Expiration makes finalization available but does not prevent additional vote calls before the permissionless state transition is submitted.
**Impact:** The settled result can include votes cast after the recorded deadline. The project considers the pre-vote stage and continued participation until finalization part of its intended governance model.
**Evidence:** MoC-Gobernanza/contracts/VotingMachine.sol:143-161,212-255; contracts/libs/VotingDataLib.sol:125-127.
**Disposition:** Rejected — no remediation planned.

### GOV-09 — Public settlement repair can override a governance stall
**Affected components:** Legacy main-RBTC-contract MoCSettlement.
**Summary:** The public task-pointer repair function unconditionally restores automatic settlement restart, including after governance has disabled it.
**Impact:** The on-chain governance stall is not persistent against this repair path. The affected settlement and BTCX functionality is deprecated and inactive.
**Evidence:** main-RBTC-contract MoCSettlement `fixTasksPointer` and `setSettlementToStall` paths.
**Disposition:** Rejected — no remediation planned.

### GOV-10 — Proposal execution permits reentrancy
**Affected components:** MoC Gobernanza VotingMachine and Governor.
**Summary:** `VotingMachine.acceptedStep` executes the winning proposal through `Governor.executeChange`, creating an external-call path that may permit reentrancy into other governance functionality. This may be a legitimate use case for some proposals, such as handling execution errors gracefully.
**Impact:** Reentrant proposal execution increases the review burden and complexity for technical reviewers. Governance has decided to disallow this behavior to simplify proposal security review.
**Evidence:** `VotingMachine.acceptedStep` → `Governor.executeChange` → `ChangeContract.execute`.
**Disposition:** Accepted — proposals must not rely on reentrant execution; no protocol remediation planned.

### GOV-11 — Proposal round counter could wrap after 256 proposals
**Affected components:** MoC Gobernanza VotingMachine.
**Summary:** The proposal voting-round counter was previously stored as `uint8`, limiting it to 256 rounds. Continued governance activity could eventually cause the value to wrap, resulting in undefined behavior. The limit was increased after the issue was repeatedly reported by automated reviewers.
**Impact:** Once the 256-round limit was reached, counter wrapping could interfere with proposal and voter-round tracking. While 256 rounds was expected to outlive the current governance system, the behavior was not well-defined.
**Evidence:** Historical `Vote.round` storage and its `uint8` casts in `VotingMachine`; widened to `uint96` in commit `8bd3db4`.
**Disposition:** Accepted — remediated by increasing the round capacity; no further remediation planned.

### GOV-12 — VotingMachine veto-query gas griefing
**Affected components:** MoC-Gobernanza VotingMachine and RegistryValues.
**Summary:** VetoMachine lookups and queries are wrapped in `try/catch`. A caller can provide enough gas for the external call to run out of gas while retaining enough gas for the fallback path to execute.
**Impact:** Griefing.
**Evidence:** MoC-Gobernanza/contracts/libs/RegistryValues.sol:48-57; contracts/VotingMachine.sol:348-356,378-397.
**Disposition:** Rejected — no remediation planned.

## Dapp

### DAPP-01 — Unauthenticated off-chain prices determine displayed transaction limits
**Affected components:** stable-protocol-interface-v3 off-chain pricing and exchange confirmation.
**Summary:** The configured off-chain API is preferred over on-chain values without signatures, freshness metadata, source comparison, or meaningful range validation. Its values determine displayed amounts and submitted minimum or maximum limits.
**Impact:** Incorrect API data can produce misleading quotes, unfavorable limits, excessive approvals, or systematic transaction failures.
**Evidence:** stable-protocol-interface-v3/src/hooks/useOffchainPrices.ts:15,50; src/hooks/useContractProtocolStatus.ts:65; src/components/Exchange/index.tsx:717; src/components/ConfirmOperation/index.tsx:423.
**Disposition:** Rejected — no remediation planned.

### DAPP-02 — Contract and spender identities are not independently authenticated
**Affected components:** stable-protocol-interface-v3 contract discovery, approvals, and transaction construction.
**Summary:** Build-time roots and RPC-returned addresses determine buckets, tokens, queues, spenders, and transaction destinations without a canonical allowlist, code-hash check, implementation verification, or independent source comparison.
**Impact:** Incorrect configuration or RPC data can make the interface approve or call unintended contracts while remaining internally consistent with its own simulation.
**Evidence:** stable-protocol-interface-v3/src/hooks/useReadContracts.ts:111,231,242,263; src/backend/moc-base.ts:18; src/backend/moc-core.ts:734,754.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.

### DAPP-03 — The production dependency graph contains unresolved advisories
**Affected components:** stable-protocol-interface-v3 direct and transitive dependencies.
**Summary:** The reviewed production graph reported numerous high and moderate advisories across HTTP, wallet, rendering, routing, and supporting packages. Several advisories are environment-specific or lacked a confirmed application-controlled source.
**Impact:** Reachable dependency defects can affect request integrity, rendering, navigation, credentials, or availability; each advisory requires applicability review rather than raw-count treatment.
**Evidence:** stable-protocol-interface-v3/package.json; pnpm-lock.yaml; production dependency audit recorded in the source findings.
**Disposition:** Accepted for remediation.

### DAPP-04 — Mutable release actions receive privileged credentials
**Affected components:** stable-protocol-interface-v3 release workflows.
**Summary:** Release workflows invoke third-party actions through mutable version tags, and one receives a personal access token used for release publication.
**Impact:** A changed action revision can affect credential confidentiality and release integrity according to the token’s granted scopes.
**Evidence:** stable-protocol-interface-v3/.github/workflows/roc-mainnet.yml:67,74,76.
**Disposition:** Rejected — no remediation planned. Release workflows are private.

### DAPP-05 — Release builds do not use the repository’s declared lockfile workflow
**Affected components:** stable-protocol-interface-v3 GitHub Actions and Dockerfile.
**Summary:** The repository declares pnpm and tracks a pnpm lockfile, while release paths use npm commands and reference an untracked npm lockfile.
**Impact:** Release builds can fail or resolve a dependency graph different from the reviewed and committed graph.
**Evidence:** stable-protocol-interface-v3/package.json:93; pnpm-lock.yaml:1; .github/workflows/roc-mainnet.yml:49; Dockerfile:7.
**Disposition:** Accepted for remediation.

### DAPP-06 — Invalid mixed-case destination checksums remain sendable
**Affected components:** stable-protocol-interface-v3 send and confirmation flows.
**Summary:** An invalid Rootstock checksum produces only a warning, does not invalidate the input, and the destination is lowercased before submission.
**Impact:** Checksum protection does not prevent transfers to mistyped mixed-case destinations, although the confirmation screen still displays the complete address.
**Evidence:** stable-protocol-interface-v3/src/components/Send/index.tsx:112,125,137; src/components/ConfirmSend/index.tsx:44,53.
**Disposition:** Accepted for remediation.

### DAPP-07 — StableX Sepolia configuration resolves to localhost
**Affected components:** stable-protocol-interface-v3 StableX environment and chain selector.
**Summary:** The StableX configuration declares Sepolia chain ID 11155111, while the selector recognizes only Rootstock IDs and maps all other values to local chain 1337.
**Impact:** The StableX build cannot correctly select its intended network and is expected to fail its network guard or remain unavailable.
**Evidence:** stable-protocol-interface-v3/.env.stablexTestnet:10; src/constants/chain.ts:19,25.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.

### DAPP-08 — Unlimited approvals increase exposure to spender changes
**Affected components:** stable-protocol-interface-v3 allowance flow.
**Summary:** The interface offers approval of the maximum uint256 value rather than limiting authorization to the immediate operation.
**Impact:** Any later spender compromise, upgrade, or address-substitution issue can affect the user’s entire approved balance rather than the intended transaction amount.
**Evidence:** stable-protocol-interface-v3/src/components/Allowance/index.tsx:26,97.
**Disposition:** Accepted for remediation.

### DAPP-09 — Lending CI exposes a deploy key to mutable action code
**Affected components:** lending-and-borrowing-sc GitHub Actions.
**Summary:** A private dependency key is available at job scope and written to the runner before an unpinned major-version third-party action executes.
**Impact:** Workflow dependency integrity and the confidentiality of private repository credentials depend on a mutable action reference.
**Evidence:** lending-and-borrowing-sc/.github/workflows/ci.yml; `pnpm/action-setup@v4` usage recorded in the source finding.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.

### DAPP-10 — Lending CI keeps deploy credentials available to repository-controlled scripts
**Affected components:** lending-and-borrowing-sc GitHub Actions and package scripts.
**Summary:** The workflow makes the private deploy key available before checkout and retains it while dependency lifecycle, lint, and test scripts execute; SSH host verification is also disabled.
**Impact:** Branch-controlled or dependency-controlled workflow code can access the key, and private dependency retrieval lacks normal host-authentication protection.
**Evidence:** lending-and-borrowing-sc/.github/workflows/ci.yml; package.json.
**Disposition:** Rejected — no remediation planned. Classified as a configuration issue.
