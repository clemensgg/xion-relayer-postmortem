# Relayer Post Mortem: Nomos ICA test - XION testnet 

**Blockers**  

- RPC query response times due to channel state bloat
- neither `hermes` no `rly` seem to be able to deal with the channel state bloat and successfully close channels.  

**Action Items**  

- [x] discuss results with `hermes` team
- [ ] feature request for `hermes` to batch channel handshakes
- [ ] possible feature request for `hermes`: only query a batch of channels (e.g. 100), try to complete handshakes, then query the next batch
- [ ] 2nd testrun: New testnet with even more nodes & `rly` instances

## Events

- `2024/04/25 14:00:00 UTC`: Public testing begins on [xion.nomos.ms](https://xion.nomos.ms)
- `2024/04/25 15:30:00 UTC`: CC Resolves gas limit config issue on relayers
- `2024/04/25 16:00:00 UTC`: Increased number of relayer instances to 4 (total)

| Status     | Value  |
|------------|--------|
| ICA-INIT   | 1500   | 
| ICA-OPEN   | ?      |
| ICA-CLOSED | ?      |

- `2024/04/25 20:00:00 UTC`: Increased number of relayer instances to 10 (total) and RPC instances to 2 (each side)  

| Status     | Value  |
|------------|--------|
| ICA-INIT   | 3800   | 
| ICA-OPEN   | 700    |
| ICA-CLOSED | ?      |

- `2024/04/26 11:30:00 UTC`: Note: At this point 10 instances of hermes seemed to be able to keep up with the channel handshakes

| Status     | Value  |
|------------|--------|
| ICA-INIT   | 208    | 
| ICA-OPEN   | 5487   |
| ICA-CLOSED | 221    |

- `2024/04/26 14:30:00 UTC`: Traffic surge

| Status     | Value  |
|------------|--------|
| ICA-INIT   | 3241   | 
| ICA-OPEN   | 6141   |
| ICA-CLOSED | 224    |

FE is reporting 52,000 unique visitors in the past 24h

- `2024/04/26 17:30:00 UTC`: Relayers get slower and slower due to growing channel state, can't keep up with traffic. Starting tests with `go-rly`, Running `hermes` profiling tests

| Status     | Value  |
|------------|--------|
| ICA-INIT   | 7809   | 
| ICA-OPEN   | 7329   |
| ICA-CLOSED | 256    |

- `2024/04/27 15:00:00 UTC`: It is at this point where the relayers really start to struggle with the state bloat. Pending channel handshakes aren't completed anymore, but some new channels in INIT state are still completed. Scaled `go-rly` to 2. Due to the massive amount of channel state hermes crashes upon startup with channel_workers enabled.

| Status     | Value  |
|------------|--------|
| ICA-INIT   | 30971  | 
| ICA-OPEN   | 13951  |
| ICA-CLOSED | 546    |

- `2024/05/2 18:00:00 UTC`: Tests confinue throughout the weekend and the beginning of May

| Status     | Value  |
|------------|--------|
| ICA-INIT   | 205662 | 
| ICA-OPEN   | 17484  |
| ICA-CLOSED | 972    |

At this point the FE is closed and public tests are stopped to give relayers the chance to deal with the channel handshake backlog.  

## Test Setup

- Number of instances `hermes_channel_worker`: 1 / 2 / 8 / 8
- Number of instances `hermes_packet_worker`: 1 / 2 / 2 / 2
- Number of instances `rly`: 0 / 0 / 1 / 2
- Number of instances `xiontestnet_node`: 1 / 1 / 2 / 2
- Number of instances `injectivetestnet_node`: 1 / 1 / 2 / 2

## Issues

- `hermes` isn't optimized for channel handshakes (no batching)
- `rly` has excessive querying / isn't performance optimized (probably can get a bit more RPC stableness out of config but RPC is generally less stable)
- injectivetestnet is comparably fragile, nodes often falling behind with 1-2 relayer instances
- hitting global file limits, OOM, RPC/gRPC response lenght limit, RPC/gRPC response timeout, block gas limit 
- DOSed injectivetestnet

## Notes

- `rly` successfully batches channel handshakes: https://testnet.explorer.injective.network/transaction/5F3837871E5B18F58F61BDACCD3228BBEB3CF9806072785C7A0C6073916BB440/
- `hermes` doesn't scan channels on startup when channel workers aren't active

## Relayer / RPC latency profiling

- profiled samples of `hermes` instances with `packet_workers` and `channel_workers` enabled
- profiles table: [hermes_profile.txt](hermes_profile.txt)

- Latency `injective-888`  
![Latency `injective-888`](output/injective_latency.png)
- Counts `injective-888`  
![Counts `injective-888`](output/injective_counts.png)
- Latency `xion-testnet-1`  
![Latency `xion-testnet-1`](output/xion_latency.png)
- Counts `xion-testnet-1`  
![Counts `xion-testnet-1`](output/xion_counts.png)
- Latency `chain_undefined`  
![Latency `chain_undefined`](output/nochain_latency.png)
- Counts `chain_undefined`  
![Counts `chain_undefined`](output/nochain_counts.png)

## Errors

### hermes

- `hermes` memory overflow:
```
2024-05-01T13:15:50.268991Z  INFO ThreadId(01) spawn:chain{chain=injective-888}:client{client=07-tendermint-239}:connection{connection=connection-220}: done spawning channel workers chain=injective-888 channel=channel-46988 
2024-05-01T13:15:50.269002Z  INFO ThreadId(01) spawn:chain{chain=injective-888}:client{client=07-tendermint-239}:connection{connection=connection-220}:channel{channel=channel-46989}: channel is TRYOPEN, state on destination chain is INIT chain=injective-888 counterparty_chain=xion-testnet-1 channel=channel-46989 
thread '<unnamed>' panicked at 'failed to set up alternative stack guard page: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:147:13 
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace 
thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29 
thread '<unnamed>' panicked at 'panic in a function that cannot unwind', library/core/src/panicking.rs:126:5 
stack backtrace: 
   0:     0x55aeda790d41 - <unknown> 
   1:     0x55aeda7be5ff - <unknown> 
   2:     0x55aeda78c6b7 - <unknown> 
   3:     0x55aeda790b55 - <unknown> 
   4:     0x55aeda7924a3 - <unknown> 
   5:     0x55aeda792234 - <unknown> 
   6:     0x55aeda792a29 - <unknown> 
   7:     0x55aeda7928e1 - <unknown> 
   8:     0x55aeda7911a6 - <unknown> 
   9:     0x55aeda792672 - <unknown> 
  10:     0x55aed8f27ad3 - <unknown> 
  11:     0x55aed8f27b77 - <unknown> 
  12:     0x55aed8f27c13 - <unknown> 
  13:     0x55aeda795952 - <unknown> 
  14:     0x7f8d02b6cac3 - <unknown> 
  15:     0x7f8d02bfe850 - <unknown> 
  16:                0x0 - <unknown> 
thread caused non-unwinding panic. aborting. 
hermes-burnt-wildcard.service: Main process exited, code=killed, status=6/ABRT 
hermes-burnt-wildcard.service: Failed with result 'signal'. 
hermes-burnt-wildcard.service: Consumed 3min 8.339s CPU time. 
hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 7. 
```
- `hermes` dies with `101/n/a`:
```
hermes-channel.service: Main process exited, code=exited, status=101/n/a 
hermes-channel.service: Failed with result 'exit-code'. 
hermes-channel.service: Consumed 8.870s CPU time. 
```
- `hermes` crashes on startup during channel scans (log level: error)
```
May 07 20:11:20 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 10678982.
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.793783Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-100}:connection{connection=connection-41}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-41
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.793965Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-101}:connection{connection=connection-42}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-42
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.799902Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-105}:connection{connection=connection-43}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-43
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.803176Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-106}:connection{connection=connection-44}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-44
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.803405Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-107}:connection{connection=connection-45}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-45
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.806158Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-113}:connection{connection=connection-49}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-49
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.807582Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-114}:connection{connection=connection-50}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-50
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.807717Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-116}:connection{connection=connection-52}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-52
May 07 22:08:05 TESTNET-RELAYER hermes[1646718]: 2024-05-07T22:08:05.807913Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-119}:connection{connection=connection-55}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-55
May 07 22:08:06 TESTNET-RELAYER hermes[1646718]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 07 22:08:06 TESTNET-RELAYER hermes[1646718]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 07 22:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 07 22:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 07 22:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 1min 23.908s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 07 22:08:11 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 9.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 07 22:08:11 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 11254207 and the job result is done.
May 07 22:08:11 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 1min 23.908s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 07 22:08:11 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 11254207.
May 07 22:08:11 TESTNET-RELAYER hermes[2126613]: 2024-05-07T22:08:11.671947Z ERROR ThreadId(01) health_check{chain=injective-888}: skipping health check, reason: failed to spawn chain runtime with error: relayer error: RPC error to endpoint http://127.0.0.1:2221/: HTTP error: error sending request for url (http://127.0.0.1:2221/): error trying to connect: tcp connect error: Connection refused (os error 111)
May 07 22:09:02 TESTNET-RELAYER hermes[2126613]: 2024-05-07T22:09:02.226473Z ERROR ThreadId(01) scan.chain{chain=xion-testnet-1}:scan.client{client=07-tendermint-72}:scan.connection{connection=connection-27}: failed to fetch connection channels: query: gRPC call `query_connection_channels` failed with status: status: Unknown, message: "transport error", details: [], metadata: MetadataMap { headers: {} }
May 07 22:10:51 TESTNET-RELAYER hermes[2126613]: 2024-05-07T22:10:51.547247Z ERROR ThreadId(01) spawn: failed to spawn worker for a chain, reason: query: error in underlying transport when making gRPC call: transport error
May 07 22:10:52 TESTNET-RELAYER hermes[2126613]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 07 22:10:52 TESTNET-RELAYER hermes[2126613]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 07 22:10:53 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 07 22:10:53 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 07 22:10:53 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 23.429s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 07 22:10:56 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 10.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 07 22:10:56 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 11268172 and the job result is done.
May 07 22:10:56 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 23.429s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 07 22:10:56 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 11268172.
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.927036Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-100}:connection{connection=connection-41}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-41
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.927201Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-101}:connection{connection=connection-42}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-42
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.943092Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-105}:connection{connection=connection-43}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-43
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.946426Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-106}:connection{connection=connection-44}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-44
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.946730Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-107}:connection{connection=connection-45}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-45
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.949462Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-113}:connection{connection=connection-49}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-49
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.950996Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-114}:connection{connection=connection-50}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-50
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.951153Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-116}:connection{connection=connection-52}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-52
May 08 00:08:04 TESTNET-RELAYER hermes[2153804]: 2024-05-08T00:08:04.951331Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-119}:connection{connection=connection-55}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-55
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: thread '<unnamed>' panicked at 'failed to set up alternative stack guard page: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:147:13
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: thread '<unnamed>' panicked at 'failed to allocate an alternative stack: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rsthread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: :143:13
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: thread '<unnamed>' panicked at 'panic in a function that cannot unwind', library/core/src/panicking.rs:126:5
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: stack backtrace:
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: thread '<unnamed>' panicked at 'panic in a function that cannot unwind', library/core/src/panicking.rs:126:5
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    0:     0x556ef26e6131 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    1:     0x556ef27139ef - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    2:     0x556ef26e1aa7 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    3:     0x556ef26e5f45 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    4:     0x556ef26e7893 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    5:     0x556ef26e7624 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    6:     0x556ef26e7e19 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    7:     0x556ef26e7cd1 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    8:     0x556ef26e6596 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:    9:     0x556ef26e7a62 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:   10:     0x556ef0e2e213 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:   11:     0x556ef0e2e2b7 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:   12:     0x556ef0e2e353 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:   13:     0x556ef26ead42 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:   14:     0x7ff112c71ac3 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:   15:     0x7ff112d03850 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]:   16:                0x0 - <unknown>
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: thread caused non-unwinding panic. aborting.
May 08 00:08:05 TESTNET-RELAYER hermes[2153804]: stack backtrace:
May 08 00:08:06 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=killed, status=6/ABRT
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'killed' and its exit status is 6.
May 08 00:08:06 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'signal'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'signal'.
May 08 00:08:06 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 2.895s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 00:08:09 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 11.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 00:08:09 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 11845662 and the job result is done.
May 08 00:08:09 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 2.895s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 00:08:09 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 11845662.
May 08 00:08:09 TESTNET-RELAYER hermes[2635854]: 2024-05-08T00:08:09.175059Z ERROR ThreadId(01) health_check{chain=injective-888}: skipping health check, reason: failed to spawn chain runtime with error: relayer error: RPC error to endpoint http://127.0.0.1:2221/: HTTP error: error sending request for url (http://127.0.0.1:2221/): error trying to connect: tcp connect error: Connection refused (os error 111)
May 08 00:09:01 TESTNET-RELAYER hermes[2635854]: 2024-05-08T00:09:01.586450Z ERROR ThreadId(01) scan.chain{chain=xion-testnet-1}:scan.client{client=07-tendermint-72}:scan.connection{connection=connection-27}: failed to fetch connection channels: query: gRPC call `query_connection_channels` failed with status: status: Unknown, message: "transport error", details: [], metadata: MetadataMap { headers: {} }
May 08 00:13:21 TESTNET-RELAYER hermes[2635854]: 2024-05-08T00:13:21.422483Z ERROR ThreadId(01) spawn: failed to spawn worker for a chain, reason: query: error in underlying transport when making gRPC call: transport error
May 08 00:13:22 TESTNET-RELAYER hermes[2635854]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 00:13:22 TESTNET-RELAYER hermes[2635854]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 00:13:23 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 08 00:13:23 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 08 00:13:23 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 23.018s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 00:13:26 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 12.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 00:13:26 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 11872396 and the job result is done.
May 08 00:13:26 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 23.018s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 00:13:26 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 11872396.
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.199591Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-100}:connection{connection=connection-41}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-41
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.199839Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-101}:connection{connection=connection-42}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-42
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.209681Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-105}:connection{connection=connection-43}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-43
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.213224Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-106}:connection{connection=connection-44}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-44
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.213453Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-107}:connection{connection=connection-45}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-45
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.216718Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-113}:connection{connection=connection-49}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-49
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.218441Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-114}:connection{connection=connection-50}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-50
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.218563Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-116}:connection{connection=connection-52}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-52
May 08 02:08:04 TESTNET-RELAYER hermes[2673295]: 2024-05-08T02:08:04.218714Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-119}:connection{connection=connection-55}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-55
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: thread '<unnamed>' panicked at 'failed to set up alternative stack guard page: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:147:13
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: thread '<unnamed>thread '' panicked at 'failed to allocate an alternative stack: Cannot allocate memory (os error 12)main', ' panicked at 'library/std/src/sys/unix/stack_overflow.rsfailed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }:', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs143::686:1329
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: thread '<unnamed>' panicked at 'thread 'panic in a function that cannot unwind<unnamed>', ' panicked at 'library/core/src/panicking.rspanic in a function that cannot unwind:', 126library/core/src/panicking.rs::5126
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: :5
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: stack backtrace:
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    0:     0x55fa90e37131 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    1:     0x55fa90e649ef - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    2:     0x55fa90e32aa7 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    3:     0x55fa90e36f45 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    4:     0x55fa90e38893 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    5:     0x55fa90e38624 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    6:     0x55fa90e38e19 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    7:     0x55fa90e38cd1 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    8:     0x55fa90e37596 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:    9:     0x55fa90e38a62 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:   10:     0x55fa8f57f213 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:   11:     0x55fa8f57f2b7 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:   12:     0x55fa8f57f353 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:   13:     0x55fa90e3bd42 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:   14:     0x7f9b5fc2eac3 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:   15:     0x7f9b5fcc0850 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]:   16:                0x0 - <unknown>
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: thread caused non-unwinding panic. aborting.
May 08 02:08:05 TESTNET-RELAYER hermes[2673295]: stack backtrace:
May 08 02:08:05 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=killed, status=6/ABRT
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'killed' and its exit status is 6.
May 08 02:08:05 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'signal'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'signal'.
May 08 02:08:05 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 59min 27.441s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 02:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 13.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 02:08:08 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 12437114 and the job result is done.
May 08 02:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 59min 27.441s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 02:08:08 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 12437114.
May 08 02:08:08 TESTNET-RELAYER hermes[3145738]: 2024-05-08T02:08:08.427221Z ERROR ThreadId(01) health_check{chain=injective-888}: skipping health check, reason: failed to spawn chain runtime with error: relayer error: RPC error to endpoint http://127.0.0.1:2221/: HTTP error: error sending request for url (http://127.0.0.1:2221/): error trying to connect: tcp connect error: Connection refused (os error 111)
May 08 02:09:02 TESTNET-RELAYER hermes[3145738]: 2024-05-08T02:09:02.069381Z ERROR ThreadId(01) scan.chain{chain=xion-testnet-1}:scan.client{client=07-tendermint-72}:scan.connection{connection=connection-27}: failed to fetch connection channels: query: gRPC call `query_connection_channels` failed with status: status: Unknown, message: "transport error", details: [], metadata: MetadataMap { headers: {} }
May 08 02:11:06 TESTNET-RELAYER hermes[3145738]: 2024-05-08T02:11:06.276719Z ERROR ThreadId(01) spawn: failed to spawn worker for a chain, reason: query: error in underlying transport when making gRPC call: transport error
May 08 02:11:07 TESTNET-RELAYER hermes[3145738]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 02:11:07 TESTNET-RELAYER hermes[3145738]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 02:11:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 08 02:11:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 08 02:11:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.791s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 02:11:11 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 14.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 02:11:11 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 12452542 and the job result is done.
May 08 02:11:11 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.791s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 02:11:11 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 12452542.
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.183755Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-100}:connection{connection=connection-41}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-41
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.183909Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-101}:connection{connection=connection-42}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-42
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.198700Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-105}:connection{connection=connection-43}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-43
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.202262Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-106}:connection{connection=connection-44}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-44
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.202532Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-107}:connection{connection=connection-45}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-45
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.205330Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-113}:connection{connection=connection-49}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-49
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.206819Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-114}:connection{connection=connection-50}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-50
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.206981Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-116}:connection{connection=connection-52}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-52
May 08 04:08:06 TESTNET-RELAYER hermes[3174315]: 2024-05-08T04:08:06.207166Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-119}:connection{connection=connection-55}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-55
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: thread '<unnamed>' panicked at 'failed to set up alternative stack guard page: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:147:13
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: thread '<unnamed>' panicked at 'failed to allocate an alternative stack: Cannot allocate memory (os error 12)thread '', mainlibrary/std/src/sys/unix/stack_overflow.rs' panicked at ':failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }143', :/rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs13:
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: 686:29
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: thread '<unnamed>' panicked at 'panic in a function that cannot unwind', library/core/src/panicking.rs:126:5
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: thread 'stack backtrace:
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: <unnamed>' panicked at 'panic in a function that cannot unwind', library/core/src/panicking.rs:126:5
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    0:     0x55c442450131 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    1:     0x55c44247d9ef - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    2:     0x55c44244baa7 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    3:     0x55c44244ff45 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    4:     0x55c442451893 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    5:     0x55c442451624 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    6:     0x55c442451e19 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    7:     0x55c442451cd1 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    8:     0x55c442450596 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:    9:     0x55c442451a62 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:   10:     0x55c440b98213 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:   11:     0x55c440b982b7 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:   12:     0x55c440b98353 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:   13:     0x55c442454d42 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:   14:     0x7fb119621ac3 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:   15:     0x7fb1196b3850 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]:   16:                0x0 - <unknown>
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: thread caused non-unwinding panic. aborting.
May 08 04:08:07 TESTNET-RELAYER hermes[3174315]: stack backtrace:
May 08 04:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=killed, status=6/ABRT
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'killed' and its exit status is 6.
May 08 04:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'signal'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'signal'.
May 08 04:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 1min 58.445s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 04:08:10 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 15.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 04:08:10 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13028565 and the job result is done.
May 08 04:08:10 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 1min 58.445s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 04:08:10 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13028565.
May 08 04:08:10 TESTNET-RELAYER hermes[3655765]: 2024-05-08T04:08:10.421674Z ERROR ThreadId(01) health_check{chain=injective-888}: skipping health check, reason: failed to spawn chain runtime with error: relayer error: RPC error to endpoint http://127.0.0.1:2221/: HTTP error: error sending request for url (http://127.0.0.1:2221/): error trying to connect: tcp connect error: Connection refused (os error 111)
May 08 04:09:01 TESTNET-RELAYER hermes[3655765]: 2024-05-08T04:09:01.655552Z ERROR ThreadId(01) scan.chain{chain=xion-testnet-1}:scan.client{client=07-tendermint-72}:scan.connection{connection=connection-27}: failed to fetch connection channels: query: gRPC call `query_connection_channels` failed with status: status: Unknown, message: "transport error", details: [], metadata: MetadataMap { headers: {} }
May 08 04:11:10 TESTNET-RELAYER hermes[3655765]: 2024-05-08T04:11:10.630034Z ERROR ThreadId(01) spawn: failed to spawn worker for a chain, reason: query: error in underlying transport when making gRPC call: transport error
May 08 04:11:11 TESTNET-RELAYER hermes[3655765]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 04:11:11 TESTNET-RELAYER hermes[3655765]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 04:11:12 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 08 04:11:12 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 08 04:11:12 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.962s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 04:11:15 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 16.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 04:11:15 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13044259 and the job result is done.
May 08 04:11:15 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.962s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 04:11:15 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13044259.
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.507211Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-100}:connection{connection=connection-41}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-41
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.507416Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-101}:connection{connection=connection-42}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-42
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.512193Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-105}:connection{connection=connection-43}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-43
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.515822Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-106}:connection{connection=connection-44}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-44
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.516114Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-107}:connection{connection=connection-45}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-45
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.519039Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-113}:connection{connection=connection-49}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-49
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.520585Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-114}:connection{connection=connection-50}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-50
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.520782Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-116}:connection{connection=connection-52}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-52
May 08 06:08:05 TESTNET-RELAYER hermes[3684353]: 2024-05-08T06:08:05.520992Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-119}:connection{connection=connection-55}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-55
May 08 06:08:06 TESTNET-RELAYER hermes[3684353]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 06:08:06 TESTNET-RELAYER hermes[3684353]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 06:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 08 06:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 08 06:08:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 41.880s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 06:08:11 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 17.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 06:08:11 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13374631 and the job result is done.
May 08 06:08:11 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 41.880s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 06:08:11 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13374631.
May 08 06:08:11 TESTNET-RELAYER hermes[3939931]: 2024-05-08T06:08:11.678486Z ERROR ThreadId(01) health_check{chain=injective-888}: skipping health check, reason: failed to spawn chain runtime with error: relayer error: RPC error to endpoint http://127.0.0.1:2221/: HTTP error: error sending request for url (http://127.0.0.1:2221/): error trying to connect: tcp connect error: Connection refused (os error 111)
May 08 06:09:02 TESTNET-RELAYER hermes[3939931]: 2024-05-08T06:09:02.209670Z ERROR ThreadId(01) scan.chain{chain=xion-testnet-1}:scan.client{client=07-tendermint-72}:scan.connection{connection=connection-27}: failed to fetch connection channels: query: gRPC call `query_connection_channels` failed with status: status: Unknown, message: "transport error", details: [], metadata: MetadataMap { headers: {} }
May 08 06:11:00 TESTNET-RELAYER hermes[3939931]: 2024-05-08T06:11:00.741995Z ERROR ThreadId(01) spawn: failed to spawn worker for a chain, reason: query: error in underlying transport when making gRPC call: transport error
May 08 06:11:01 TESTNET-RELAYER hermes[3939931]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 06:11:01 TESTNET-RELAYER hermes[3939931]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 06:11:02 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 08 06:11:02 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 08 06:11:02 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 23.427s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 06:11:05 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 18.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 06:11:05 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13382212 and the job result is done.
May 08 06:11:05 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 23.427s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 06:11:05 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13382212.
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.257516Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-100}:connection{connection=connection-41}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-41
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.257692Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-101}:connection{connection=connection-42}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-42
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.264140Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-105}:connection{connection=connection-43}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-43
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.267698Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-106}:connection{connection=connection-44}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-44
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.267941Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-107}:connection{connection=connection-45}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-45
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.270673Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-113}:connection{connection=connection-49}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-49
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.272094Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-114}:connection{connection=connection-50}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-50
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.272222Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-116}:connection{connection=connection-52}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-52
May 08 08:08:06 TESTNET-RELAYER hermes[3961197]: 2024-05-08T08:08:06.272383Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-119}:connection{connection=connection-55}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-55
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: thread '<unnamed>' panicked at 'failed to set up alternative stack guard page: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:147:13
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: thread '<unnamed>' panicked at 'failed to allocate an alternative stack: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:thread '143:main13' panicked at '
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: thread '<unnamed>thread '' panicked at '<unnamed>panic in a function that cannot unwind' panicked at '', panic in a function that cannot unwindlibrary/core/src/panicking.rs', :library/core/src/panicking.rs126::1265:
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: 5
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: stack backtrace:
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    0:     0x55da90048131 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    1:     0x55da900759ef - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    2:     0x55da90043aa7 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    3:     0x55da90047f45 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    4:     0x55da90049893 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    5:     0x55da90049624 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    6:     0x55da90049e19 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    7:     0x55da90049cd1 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    8:     0x55da90048596 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:    9:     0x55da90049a62 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:   10:     0x55da8e790213 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:   11:     0x55da8e7902b7 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:   12:     0x55da8e790353 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:   13:     0x55da9004cd42 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:   14:     0x7f64eccd0ac3 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:   15:     0x7f64ecd62850 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]:   16:                0x0 - <unknown>
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: thread caused non-unwinding panic. aborting.
May 08 08:08:07 TESTNET-RELAYER hermes[3961197]: stack backtrace:
May 08 08:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=killed, status=6/ABRT
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'killed' and its exit status is 6.
May 08 08:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'signal'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'signal'.
May 08 08:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 2min 15.921s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 08:08:10 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 19.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 08:08:10 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13671491 and the job result is done.
May 08 08:08:10 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 2min 15.921s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 08:08:10 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13671491.
May 08 08:08:10 TESTNET-RELAYER hermes[4178042]: 2024-05-08T08:08:10.669920Z ERROR ThreadId(01) health_check{chain=injective-888}: skipping health check, reason: failed to spawn chain runtime with error: relayer error: RPC error to endpoint http://127.0.0.1:2221/: HTTP error: error sending request for url (http://127.0.0.1:2221/): error trying to connect: tcp connect error: Connection refused (os error 111)
May 08 08:11:22 TESTNET-RELAYER hermes[4178042]: 2024-05-08T08:11:22.219550Z ERROR ThreadId(01) spawn: failed to spawn worker for a chain, reason: query: gRPC call `query_clients` failed with status: status: Unknown, message: "transport error", details: [], metadata: MetadataMap { headers: {} }
May 08 08:11:23 TESTNET-RELAYER hermes[4178042]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 08:11:23 TESTNET-RELAYER hermes[4178042]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 08:11:24 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 08 08:11:24 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 08 08:11:24 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.925s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 08:11:27 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 20.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 08:11:27 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13680003 and the job result is done.
May 08 08:11:27 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.925s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 08:11:27 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13680003.
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.569250Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-100}:connection{connection=connection-41}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-41
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.569477Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-101}:connection{connection=connection-42}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-42
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.579251Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-105}:connection{connection=connection-43}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-43
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.582929Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-106}:connection{connection=connection-44}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-44
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.583276Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-107}:connection{connection=connection-45}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-45
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.586084Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-113}:connection{connection=connection-49}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-49
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.587530Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-114}:connection{connection=connection-50}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-50
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.587642Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-116}:connection{connection=connection-52}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-52
May 08 10:08:06 TESTNET-RELAYER hermes[6486]: 2024-05-08T10:08:06.587773Z ERROR ThreadId(01) spawn:chain{chain=xion-testnet-1}:client{client=07-tendermint-119}:connection{connection=connection-55}: skipped connection workers, reason: relayer error: error in underlying transport when making gRPC call: transport error chain=xion-testnet-1 connection=connection-55
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: thread '<unnamed>' panicked at 'failed to set up alternative stack guard page: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:147:13
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: thread '<unnamed>' panicked at 'failed to allocate an alternative stack: Cannot allocate memory (os error 12)', library/std/src/sys/unix/stack_overflow.rs:143:13
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: thread '<unnamed>' panicked at 'panic in a function that cannot unwind', library/core/src/panicking.rs:thread '126<unnamed>:5' panicked at '
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: panic in a function that cannot unwind', library/core/src/panicking.rsstack backtrace:
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: :126:5
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    0:     0x55861bea4131 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    1:     0x55861bed19ef - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    2:     0x55861be9faa7 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    3:     0x55861bea3f45 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    4:     0x55861bea5893 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    5:     0x55861bea5624 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    6:     0x55861bea5e19 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    7:     0x55861bea5cd1 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    8:     0x55861bea4596 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:    9:     0x55861bea5a62 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:   10:     0x55861a5ec213 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:   11:     0x55861a5ec2b7 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:   12:     0x55861a5ec353 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:   13:     0x55861bea8d42 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:   14:     0x7f779b275ac3 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:   15:     0x7f779b307850 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]:   16:                0x0 - <unknown>
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: thread caused non-unwinding panic. aborting.
May 08 10:08:07 TESTNET-RELAYER hermes[6486]: stack backtrace:
May 08 10:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=killed, status=6/ABRT
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'killed' and its exit status is 6.
May 08 10:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'signal'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'signal'.
May 08 10:08:07 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 1min 21.427s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 10:08:10 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 21.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 10:08:10 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13968218 and the job result is done.
May 08 10:08:10 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 1h 1min 21.427s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 10:08:10 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13968218.
May 08 10:08:10 TESTNET-RELAYER hermes[223067]: 2024-05-08T10:08:10.938800Z ERROR ThreadId(01) health_check{chain=injective-888}: skipping health check, reason: failed to spawn chain runtime with error: relayer error: RPC error to endpoint http://127.0.0.1:2221/: HTTP error: error sending request for url (http://127.0.0.1:2221/): error trying to connect: tcp connect error: Connection refused (os error 111)
May 08 10:11:06 TESTNET-RELAYER hermes[223067]: 2024-05-08T10:11:06.953901Z ERROR ThreadId(01) spawn: failed to spawn worker for a chain, reason: query: gRPC call `query_clients` failed with status: status: Unknown, message: "transport error", details: [], metadata: MetadataMap { headers: {} }
May 08 10:11:07 TESTNET-RELAYER hermes[223067]: thread 'main' panicked at 'failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }', /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/std/src/thread/mod.rs:686:29
May 08 10:11:07 TESTNET-RELAYER hermes[223067]: note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
May 08 10:11:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Main process exited, code=exited, status=101/n/a
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ An ExecStart= process belonging to unit hermes-burnt-wildcard.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 101.
May 08 10:11:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has entered the 'failed' state with result 'exit-code'.
May 08 10:11:08 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.995s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 10:11:12 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Scheduled restart job, restart counter is at 22.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ Automatic restarting of the unit hermes-burnt-wildcard.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
May 08 10:11:12 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 13976065 and the job result is done.
May 08 10:11:12 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 22.995s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 10:11:12 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 13976065.
May 08 10:33:14 TESTNET-RELAYER systemd[1]: Stopping hermes burnt...
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has begun execution
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has begun execution.
░░ 
░░ The job identifier is 14031021.
May 08 10:33:14 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Deactivated successfully.
░░ Subject: Unit succeeded
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has successfully entered the 'dead' state.
May 08 10:33:14 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 14031021 and the job result is done.
May 08 10:33:14 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Consumed 10min 36.291s CPU time.
░░ Subject: Resources consumed by unit runtime
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service completed and consumed the indicated resources.
May 08 10:33:14 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 14031021.
May 08 10:33:49 TESTNET-RELAYER systemd[1]: Stopping hermes burnt...
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has begun execution
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has begun execution.
░░ 
░░ The job identifier is 14032484.
May 08 10:33:49 TESTNET-RELAYER systemd[1]: hermes-burnt-wildcard.service: Deactivated successfully.
░░ Subject: Unit succeeded
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ The unit hermes-burnt-wildcard.service has successfully entered the 'dead' state.
May 08 10:33:49 TESTNET-RELAYER systemd[1]: Stopped hermes burnt.
░░ Subject: A stop job for unit hermes-burnt-wildcard.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A stop job for unit hermes-burnt-wildcard.service has finished.
░░ 
░░ The job identifier is 14032484 and the job result is done.
May 08 10:33:49 TESTNET-RELAYER systemd[1]: Started hermes burnt.
░░ Subject: A start job for unit hermes-burnt-wildcard.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░ 
░░ A start job for unit hermes-burnt-wildcard.service has finished successfully.
░░ 
░░ The job identifier is 14032484.
```

### go-rly

- `rly`'s query pressure is generally so high that the injective node immediately starts to fall back
```
rpc error: code = Unknown desc = failed to execute message; message index: 0: failed to verify header: invalid header: new header has a time from the future 2024-05-01 12:31:05.687617525 +0000 UTC (now: 2024-05-01 12:29:52.508240025 +0000 UTC; max clock drift: 40s)
```
- `rly` file limit overflow:
```
Flush not complete        {"error": "failed to query packet commitments: post failed: Post \"http://127.0.0.1:2131\": dial tcp 127.0.0.1:2131: socket: too many open files"}
```

### Xion fullnode

- `xiond` gRPC response length overflow response in `hermes`:
```
2024-05-01T14:28:22.725182Z ERROR ThreadId(01) scan.chain{chain=xion-testnet-1}:scan.client{client=07-tendermint-119}:scan.connection{connection=connection-55}: failed to fetch connection channels: query: gRPC call `query_connection_channels` failed with status: status: OutOfRange, message: "Error, message length too large: found 59542869 bytes, the limit is: 33554432 bytes", details: [], metadata: MetadataMap { headers: {"content-type": "application/grpc", "x-cosmos-block-height": "7662623"} } 
```
^ bumped gRPC limits in `app.toml`:
```
# MaxRecvMsgSize defines the max message size in bytes the server can receive.
# The default value is 10MB.
# default value
# max-recv-msg-size = "10485760"
# increased x10 for xiontestnet
max-recv-msg-size = "104857600"

# MaxSendMsgSize defines the max message size in bytes the server can send.
# The default value is math.MaxInt32.
# default value
# max-send-msg-size = "2147483647"
# increased x10 for xiontestnet
max-send-msg-size = "21474836470"
```
^ set `max_grpc_decoding_size` in hermes `config.toml`:
```
max_grpc_decoding_size = 648251000
```


