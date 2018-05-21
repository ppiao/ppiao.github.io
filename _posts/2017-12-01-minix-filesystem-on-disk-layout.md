---
layout: post
title: Minix Filesystem On-Disk Layout
---

I recommend you to read Operating Systems: Three Easy Pieces: Chapter 40:
File System Implementation. Most knowledge from this book. Here I reference
some Linux kernel code and write a userspace toy software to make it easy to
understand.

> > What types of on-disk structures are utilized by the file system to organize
> > its data and metadata?
> >
> > How does it map the calls made by a process, such as open(), read(), write(),
> > etc., onto its structures?
>
> To think about file systems, we usually suggest thinking about these two
> different aspects of them.

# 1. Divide disk into blocks

![](/images/divide-disk-into-blocks.png)

# 2. Divide disk into different structures

![](/images/filesystem-layout.png)

	+-----+-----------------+-----------------+-----------------+-----------+----+
	|boot |minix_super_block|    inode map    |     zone map    |minix_inode|data|
	+-----+-----------------+-----------------+-----------------+-----------+----+
	|--1--|--------1--------|sb->s_imap_blocks|sb->s_zmap_blocks|
	Number of minix_inode: sb->s_ninodes.
	Number of data zone: sb->s_nzones.

	Directory entry layout:
	+----------------------+----------------------+
	|minix_dir_entry + NAME|minix_dir_entry + NAME|
	+----------------------+----------------------+
	Each entry length is: sizeof(struct minix_dir_entry) + name length.

# 3. Read minix_super_block

	static void print_minix_info(const char *dev)
	{
		...
		if ((fd = open(dev, O_RDONLY)) == -1) {
			perror("open");
			exit(EXIT_FAILURE);
		}
		...
	}

	static void print_minix_super_block(const int fd, struct minix_super_block *sb)
	{
		if (lseek(fd, 1 * BLOCK_SIZE, SEEK_SET) == -1) {
			perror("lseek");
			exit(EXIT_FAILURE);
		}
		if (read(fd, sb, sizeof(*sb)) == -1) {
			perror("read");
			exit(EXIT_FAILURE);
		}
		...
	}

According to Minix filesystem layout, the offset struct minix_super_block is
BLOCK_SIZE. So reposition the offset to BLOCK_SIZE, we can read
minix_super_block.

	/*
	 * minix super-block data on disk
	 */
	struct minix_super_block {
		__u16 s_ninodes;
		__u16 s_nzones;
		__u16 s_imap_blocks;
		__u16 s_zmap_blocks;
		__u16 s_firstdatazone;
		__u16 s_log_zone_size;
		__u32 s_max_size;
		__u16 s_magic;
		__u16 s_state;
		__u32 s_zones;
	};

# 4. Count free inode and zone

	static void print_bitmap(const int fd, const struct minix_super_block *sb)
	{
		unsigned char block[BLOCK_SIZE];
		int i, j, count = 0;

		if (lseek(fd, 2 * BLOCK_SIZE, SEEK_SET) == -1) {
			perror("lseek");
			exit(EXIT_FAILURE);
		}
		for (i = 0; i < sb->s_imap_blocks; i++) {
			if (read(fd, block, sizeof(block)) == -1) {
				perror("read");
				exit(EXIT_FAILURE);
			}
			for (j = 0; j < BLOCK_SIZE; j++)
				count += 8 - hweight8(block[j]);
		}
		...
		count = 0;
		for (i = 0; i < sb->s_zmap_blocks; i++) {
			if (read(fd, block, sizeof(block)) == -1) {
				perror("read");
				exit(EXIT_FAILURE);
			}
			for (j = 0; j < BLOCK_SIZE; j++)
				count += 8 - hweight8(block[j]);
		}
		...
	}

The offset of inode map is 2 * BLOCK_SIZE, the number of inode map block is
sb->s_imap_blocks, zone map after inode map, and number of zone blocks is
sb->s_zmap_blocks.

# 5. Read minix_inode

	static void read_inode(const int fd, const struct minix_super_block *sb,
			       ino_t ino, struct minix_inode *inode)
	{
		off_t offset = (2 + sb->s_imap_blocks + sb->s_zmap_blocks) * BLOCK_SIZE
			       + (ino - 1) * sizeof(*inode);

		if (lseek(fd, offset, SEEK_SET) == -1) {
			perror("lseek");
			exit(EXIT_FAILURE);
		}
		if (read(fd, inode, sizeof(*inode)) == -1) {
			perror("read");
			exit(EXIT_FAILURE);
		}
	}

The offset of minix_inode is (2 + sb->s_imap_blocks + sb->s_zmap_blocks). This
is the address of a minix_inode array.

# 6. Print the regular file

	static void read_file(const int fd, const struct minix_inode *inode)
	{
		unsigned char block[BLOCK_SIZE];
		int i;

		/* Only print direct data block */
		for (i = 0; i <= 7; i++) {
			if (!inode->i_zone[i])
				break;
			if (lseek(fd, inode->i_zone[i] * BLOCK_SIZE, SEEK_SET) == -1) {
				perror("lseek");
				exit(EXIT_FAILURE);
			}
			if (read(fd, block, sizeof(block)) == -1) {
				perror("read");
				exit(EXIT_FAILURE);
			}
			printf("%s", block);
		}
	}

inode->i_zone array recording block number of the file. I only print direct data
block.

# 7. Print the directory

	static void read_dir(const int fd, const struct minix_inode *inode)
	{
		struct minix_dir_entry *dentry;
		unsigned char block[BLOCK_SIZE];
		int i, j;

		/* Only print direct data block */
		for (i = 0; i <= 7; i++) {
			if (!inode->i_zone[i])
				break;
			if (lseek(fd, inode->i_zone[i] * BLOCK_SIZE, SEEK_SET) == -1) {
				perror("lseek");
				exit(EXIT_FAILURE);
			}
			if (read(fd, block, sizeof(block)) == -1) {
				perror("read");
				exit(EXIT_FAILURE);
			}
			dentry = (struct minix_dir_entry *) block;
			for (j = 0; j < BLOCK_SIZE / sizeof(*dentry); j++) {
				if (!dentry->name[0])
					continue;
				printf("%s\n", dentry->name);
				dentry = (void *) dentry + sizeof(*dentry) + 14;
			}
		}
	}

Like the regular file, inode->i_zone array recording block number.  The layout
of the directory is an array that the length is
(sizeof(struct minix_dir_entry) + NAMELENGTH).

	struct minix_dir_entry {
		__u16 inode;
		char name[0];
	};

# 8. More

A toy for learning Minix filesystem.

minixinfo: [https://github.com/ppiao/minixinfo](https://github.com/ppiao/minixinfo)

**Reference**

* Operating Systems: Three Easy Pieces
* Linux kernel 4.15
