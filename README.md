# Redis Cluster

This is a CLI tool I built that uses Ansible under the hood to manage a 6-node Redis Cluster. It handles everything from initial setup to rolling upgrades — all without any downtime.

The idea is simple: instead of SSHing into servers and running commands manually, you just use `./redis-tool` and it takes care of the rest.

---

## Getting the Infrastructure Up and Running

First things first — we need 6 containers that act like real servers. They run Ubuntu with SSH, so Ansible can connect to them just like it would in production.

### What You Need Installed

- **Docker**
- **Ansible** 2.14 or newer — installed
- **Python 3** — for the CLI tool itself

### Spinning Everything Up

# First, create an SSH key if you don't have one already
ssh-keygen -t rsa -N "" -f /home/Ansible/id_rsa

# Build the container image and start all 6 nodes
cd infra
docker compose up -d --build

# If you're using Podman instead:
# podman-compose up -d --build

# Make sure everything came up
docker ps

# Quick sanity check — can we SSH into the nodes?
for i in 1 2 3 4 5 6; do
  ssh -i /home/Ansible/id_rsa -o StrictHostKeyChecking=no root@10.10.0.1$i echo "OK"
done
The Nodes
Here's what we're working with:

Container	IP	Starts As
redis-node-1	10.10.0.11	Master
redis-node-2	10.10.0.12	Master
redis-node-3	10.10.0.13	Master
redis-node-4	10.10.0.14	Replica
redis-node-5	10.10.0.15	Replica
redis-node-6	10.10.0.16	Replica
They're all on a static 10.10.0.0/24 network so the IPs never change.

Using the Tool
Here's every command and what it actually does:

Set Up the Cluster
./redis-tool provision 7.0.15
This does quite a bit behind the scenes — it installs Redis 7.0.15 from source on all 6 nodes, writes out the cluster config, starts everything up, and then forms the cluster so you end up with 3 masters each with a replica.

See What's Going On
./redis-tool status
Gives you a quick snapshot — which nodes are masters, which are replicas, what version they're running, how many keys they hold, and how much memory they're using.

Load Some Test Data
./redis-tool data seed --keys 1000
Puts 1000 key-value pairs into the cluster. The values are just SHA256 hashes of the key names, so we can always recalculate what they should be. The keys spread across all the masters automatically thanks to hash slots.

Check the Data is Correct
./redis-tool data verify --keys 1000
Reads every key back out, recalculates what the value should be, and compares. You'll get either a nice "PASS — 1000/1000 keys verified" or it'll tell you exactly what's wrong.

Upgrade Without Downtime
./redis-tool upgrade --version 7.2.6
This is the big one. It upgrades every node from 7.0.15 to 7.2.6 without the cluster ever going down. More details on how below.

Run a Full Health Check
./redis-tool verify --full
Checks everything at once — cluster state, version consistency, slot coverage, topology, replication health, and data integrity. Basically answers the question "is everything actually working properly?"

How the Rolling Upgrade Works
This was the trickiest part to get right. The goal is simple: upgrade all 6 nodes without any moment where the cluster can't serve requests.

some pre-flight checks:

Is the cluster healthy right now?
Can we reach all 6 nodes?
Are we actually changing versions? (no point upgrading 7.2.6 to 7.2.6)
Then we upgrade the replicas, one by one:

For each replica:

Stop Redis on it
Install the new version (compile from source)
Start it back up
Wait for it to reconnect and sync with its master
Make sure the cluster is still happy before touching the next one
This is safe because replicas don't serve traffic directly — the masters handle everything.

Then the masters, one by one (this is the clever bit):

For each master:

Tell its replica to take over (CLUSTER FAILOVER) — the replica becomes the new master
Now the old master is just a replica, so it's safe to touch
Stop Redis on it
Install the new version
Start it back up — it rejoins as a replica
Verify cluster health before moving on
Finally, verify everything:

All nodes reporting the new version? ✓
Cluster state still ok? ✓
All our data still there and correct? ✓
Why Do It This Way?
I went with this approach because:

Replicas first — they're not serving traffic, so restarting them is harmless
Failover before touching masters — there's always a master available for every slot range, so clients never see an error
One at a time — if something goes wrong, we've only affected one node
Health checks between each node — if the cluster gets unhappy, we stop immediately rather than making things worse
What If Something Breaks?
If any step fails, the tool stops right there. It tells you which node had the problem and what it was trying to do. It won't keep going and potentially make things worse.

I didn't implement automatic rollback (that's a stretch goal), so if it fails mid-way, you'll have some nodes on the old version and some on the new. The cluster should still work fine in that state though — Redis handles mixed versions gracefully.

Assumptions and Trade-offs I Made
Here's my thinking on some of the decisions:

Building from source instead of using packages — This gives us exact version control. Package repos might not have the specific version we need, or might lag behind. The downside is it takes longer (compiling takes a minute or two per node).

Static IPs instead of DNS — Keeps the Ansible inventory dead simple and reliable. In a real production setup you'd probably use DNS or service discovery, but for this it's overkill.

No TLS or authentication — This is a lab environment on an isolated network. Adding TLS would complicate the setup without adding value for what we're demonstrating.

AOF persistence enabled — So data survives if Redis restarts within a container's lifetime. We don't need RDB snapshots on top of that.

Root access everywhere — These are throwaway containers, not production servers. Using root keeps things simple and avoids permission headaches.

No automatic rollback — I decided to keep the scope focused. The tool stops on failure and reports what happened, which is honestly what most teams want anyway — they'd rather investigate than have automation make more changes.

Known Limitations
A few things to be aware of:

Redis doesn't auto-start when containers restart. The containers run sshd as their main process, not Redis. If you restart a container, you'll need to start Redis again (or just re-run provision).

You can't downgrade easily. Redis 7.2.6 writes data in a format that 7.0.15 can't read. If you need to go back, you'd have to clear the data files first.

No rollback command. If an upgrade fails halfway through, you'll need to manually sort out the affected node. The cluster will still work, just with mixed versions.

Everything runs from one control node. If your host machine dies mid-upgrade, you'll need to check where things got to and potentially finish manually.

The provision command won't fix a broken cluster. If the cluster is already formed but unhealthy, running provision again won't help — it sees the version matches and skips.

Project Layout

/home/Ansible/
├── redis-tool                    ← The CLI you actually run
├── ansible/
│   ├── ansible.cfg               ← Ansible settings
│   ├── inventory/
│   │   └── hosts.ini             ← All 6 nodes defined here
│   ├── playbooks/
│   │   ├── provision.yml         ← Install + cluster formation
│   │   ├── status.yml            ← Status report
│   │   ├── seed.yml              ← Data seeding
│   │   ├── verify.yml            ← Data verification
│   │   ├── upgrade.yml           ← The rolling upgrade
│   │   └── full_verify.yml       ← Comprehensive checks
│   └── roles/
│       └── redis/
│           ├── defaults/main.yml
│           ├── tasks/
│           │   ├── main.yml
│           │   ├── install.yml   ← Compiles Redis from source
│           │   └── configure.yml ← Writes config + starts Redis
│           ├── handlers/main.yml
│           └── templates/
│               └── redis.conf.j2
├── infra/
│   ├── Containerfile             ← The container image definition
│   └── compose.yml              ← Spins up all 6 nodes
├── output/                       ← Terminal output captures
├── logs/
│   └── operation_log.json        ← JSON log of every operation
└── README.md                     ← You're reading it


If Things Go Wrong
Redis won't start (complains about RDB format):

for i in 1 2 3 4 5 6; do
  docker exec redis-node-$i bash -c "rm -rf /var/lib/redis/appendonlydir"
  docker exec redis-node-$i bash -c "rm -f /var/lib/redis/nodes_6379.conf"
  docker exec redis-node-$i redis-server /etc/redis/redis.conf --daemonize yes
done

###
Then re-form the cluster with ./redis-tool provision.

Ansible can't reach the nodes:

# Check containers are actually running
docker ps

# Check SSH works
ssh -i /home/Ansible/id_rsa -o StrictHostKeyChecking=no root@10.10.0.11 echo "OK"
Want to check things manually:

# Check cluster health manually
docker exec redis-node-1 redis-cli cluster info
docker exec redis-node-1 redis-cli cluster nodes



