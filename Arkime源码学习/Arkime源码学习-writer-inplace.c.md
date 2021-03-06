## 简介
`Writer that doesn't actually write pcap instead using location of reading
实际上并没有编写pcap的编写器，而是使用阅读地点`
## 源码
```c
#define _FILE_OFFSET_BITS 64
#include "moloch.h"
#include <errno.h>
#include <fcntl.h>
#include <inttypes.h>
#include <pthread.h>
#include <sys/stat.h>
#include <sys/mman.h>

extern MolochConfig_t        config;


LOCAL MOLOCH_LOCK_DEFINE(filePtr2Id);

extern char                *readerFileName[256];
extern uint32_t             readerOutputIds[256];

/******************************************************************************/
LOCAL uint32_t writer_inplace_queue_length()
{
    return 0;
}
/******************************************************************************/
LOCAL void writer_inplace_exit()
{
}
/******************************************************************************/
LOCAL long writer_inplace_create(MolochPacket_t * const packet)
{
    struct stat st;
    const char *readerName = readerFileName[packet->readerPos];

    stat(readerName, &st);

    uint32_t outputId;
    if (config.pcapReprocess) {
        moloch_db_file_exists(readerName, &outputId);
    } else {
        char *filename;
        if (config.gapPacketPos)
            filename = moloch_db_create_file_full(packet->ts.tv_sec, readerName, st.st_size, !config.noLockPcap, &outputId,
                                                  "packetPosEncoding", "gap0",
                                                  (char *)NULL);
        else
            filename = moloch_db_create_file(packet->ts.tv_sec, readerName, st.st_size, !config.noLockPcap, &outputId);

        g_free(filename);
    }
    readerOutputIds[packet->readerPos] = outputId;
    return outputId;
}

/******************************************************************************/
LOCAL void writer_inplace_write(const MolochSession_t * const UNUSED(session), MolochPacket_t * const packet)
{
    // Need to lock since multiple packet threads for the same readerPos are running and only want to create once
    MOLOCH_LOCK(filePtr2Id);
    long outputId = readerOutputIds[packet->readerPos];
    if (!outputId)
        outputId = writer_inplace_create(packet);
    MOLOCH_UNLOCK(filePtr2Id);

    packet->writerFileNum = outputId;
    packet->writerFilePos = packet->readerFilePos;
}
/******************************************************************************/
LOCAL void writer_inplace_write_dryrun(const MolochSession_t * const UNUSED(session), MolochPacket_t * const packet)
{
    packet->writerFilePos = packet->readerFilePos;
}
/******************************************************************************/
void writer_inplace_init(char *UNUSED(name))
{
    config.gapPacketPos        = moloch_config_boolean(NULL, "gapPacketPos", TRUE);
    moloch_writer_queue_length = writer_inplace_queue_length;
    moloch_writer_exit         = writer_inplace_exit;
    if (config.dryRun)
        moloch_writer_write    = writer_inplace_write_dryrun;
    else
        moloch_writer_write    = writer_inplace_write;
}

```