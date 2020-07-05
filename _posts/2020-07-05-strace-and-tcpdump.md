---
layout: post
title: Using strace inspect how tcpdump work
---

1. Build latest strace

        $ git clone --depth 1 https://github.com/strace/strace.git
        $ cd strace
        $ sudo apt-get install gawk automake gcc-multilib
        $ ./bootstrap
        $ ./configure
        $ make
        $ sudo make install

2. Using strace trace tcpdump

        $ sudo strace -v -e trace=setsockopt,socket tcpdump -i any icmp
        socket(AF_PACKET, SOCK_DGRAM, htons(ETH_P_ALL)) = 3
        setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, {len=6, filter=[BPF_STMT(BPF_LD|BPF_H|BPF_ABS, SKF_AD_OFF+SKF_AD_PROTOCOL), BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x800, 0, 0x3), BPF_STMT(BPF_LD|BPF_B|BPF_ABS, 0x9), BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x1, 0, 0x1), BPF_STMT(BPF_RET|BPF_K, 0x40000), BPF_STMT(BPF_RET|BPF_K, 0)]}, 16) = 0

3. How tcpdump work

    * Use socket init AF_PACKET sockfd
    * Use setsockopt attach BPF to relate sockfd
    * Read from sockfd

4. A packet capture example

        #include <stdio.h>
        #include <stdlib.h>
        #include <string.h>
        #include <unistd.h>
        #include <sys/types.h>
        #include <sys/socket.h>
        #include <arpa/inet.h>
        #include <linux/filter.h>
        #include <net/if.h>
        #include <net/ethernet.h>

        int main(int argc, char *argv[]) {
            int sockfd;
            char buf[ETHER_MAX_LEN];
            struct ether_header eh;
            struct sock_filter filter[] = {
                BPF_STMT(BPF_LD|BPF_H|BPF_ABS, 0xc),
                BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x800, 0, 0x3),
                BPF_STMT(BPF_LD|BPF_B|BPF_ABS, 0x17),
                BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x1, 0, 0x1),
                BPF_STMT(BPF_RET|BPF_K, 0x40000),
                BPF_STMT(BPF_RET|BPF_K, 0)
            };
            struct sock_fprog bpf = {
                .len = (unsigned short) (sizeof(filter) / sizeof(filter[0])),
                .filter = filter
            };

            sockfd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
            if (sockfd < 0) {
                perror("socket");
                exit(1);
            }
            if (setsockopt(sockfd, SOL_SOCKET, SO_ATTACH_FILTER, &bpf, sizeof(bpf)) < 0) {
                perror("setsockopt");
                exit(1);
            }
            if (read(sockfd, buf, sizeof(buf)) < 0) {
                perror("read");
                exit(1);
            }
            memcpy(&eh, buf, sizeof(eh));
            for (int i = 0; i < ETH_ALEN; i++) {
                printf("%s%X", i == 0 ? "" : ":", eh.ether_shost[i]);
            }
            return 0;
        }

5. What is BPF code [1]

        1 BPF_STMT(BPF_LD|BPF_H|BPF_ABS, 0xc)
        2 BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x800, 0, 0x3)
        3 BPF_STMT(BPF_LD|BPF_B|BPF_ABS, 0x17)
        4 BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x1, 0, 0x1)
        5 BPF_STMT(BPF_RET|BPF_K, 0x40000)
        6 BPF_STMT(BPF_RET|BPF_K, 0)

    When recv an ethernet frame, execute BPF code filter the frame, capture it if BPF code return 0x40000.

    Translate BPF code to C:

        int filter(struct frame f) {
            int8_t breg;
            int16_t hreg;
            int32_t wreg;

            // 1 BPF_STMT(BPF_LD|BPF_H|BPF_ABS, 0xc): load to register
            memcpy(&hreg, (char *) &f + 0xc, sizeof(hreg));
            // 2 BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x800, 0, 0x3): if true go to next line or go to fourth line
            if (hreg == 0x800) {
                goto no_capture;
            }
            // 3 BPF_STMT(BPF_LD|BPF_B|BPF_ABS, 0x17)
            memcpy(&breg, (char *) &f + 0x17, sizeof(breg));
            // 4 BPF_JUMP(BPF_JMP|BPF_K|BPF_JEQ, 0x1, 0, 0x1): if true go to next line or go to second line
            if (hreg == 0x1) {
                goto no_capture;
            }
            // 5 BPF_STMT(BPF_RET|BPF_K, 0x40000)
            return 0x40000;
        no_capture:
            // 6 BPF_STMT(BPF_RET|BPF_K, 0)
            return 0;
        }

6. More

    * tcpdump using [libpcap][libpcap] capture packet [2]
    * BPF also use in other place [3]

**links:**

* [1] [Linux Socket Filtering aka Berkeley Packet Filter (BPF)][bpf]
* [2] [libpcap - the LIBpcap interface to various kernel packet capture mechanism][libpcap]
* [3] [seccomp - SECure COMPuting with filters][seccomp]

[bpf]: https://github.com/torvalds/linux/blob/master/Documentation/networking/filter.rst
[libpcap]: https://github.com/the-tcpdump-group/libpcap
[seccomp]: https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt
