cbrecovery command usage
========================


        cbrecovery
        -n  --node-source,
                     type="string", default="127.0.0.1:8091",
                     metavar="127.0.0.1:8091",
                     help="source cluster to recover data from"

        -N  --node-destination,
                     type="string", default="127.0.0.1:8091",
                     metavar="127.0.0.1:8091",
                     help="destination cluster to recover data to"

        -b  --bucket-source
                     type="string", default="default",
                     metavar="default",
                     help="""source bucket to recover from """

        -B  --bucket-destination
                     type="string", default="default",
                     metavar="default",
                     help="""destination bucket to recover to """

        -u  --username
                     type="string", default=None,
                     help="REST username for source cluster"

        -p  --password
                     type="string", default=None,
                     help="REST password for source cluster"

        -U  --username-destination
                     type="string", default=None,
                     help="REST username for destination cluster or server node"

        -P  --password-destination
                     type="string", default=None,
                     help="REST password for destination cluster or server node"

        -l  --vbucket-list
                     type="string", default=None,
                     help="""transmit data only from specified vbuckets"""

        -v  --verbose
                     action="count", default=0,
                     help="verbose logging; more -v's provide more verbosity"

New REST API proposal for cbrecovery tool
----------------------------------------

 - Retrieve missing vbucket list

> GET
> /pools/default/buckets/bucket_name/vbucketsMissing
> [500, 612, 712]

 - Add replacement node to cluster

> POST
> /pools/default/buckets/bucket_name/replaceNode
> add=<newnode>&remove=<oldnode>

*ok*: 202

*error*:

401: node is not found

402: node exists already

403: node is still in active mode, you have to failover it first

 - set vbucket state from missing to replica

> POST
> /pools/default/buckets/bucket_name/changeVbucketStates
> vbucket=500&vbucket=612&vbucket=712&state=replica

ok: 202

error:

410: vbucket is not found

411: state is not valid, it has to be active, replica, pending or dead

 - set vbucket state from replica to active

> POST
> /pools/default/buckets/bucket_name/changeVbucketStates
> vbucket=500&vbucket=612&vbucket=712&state=active

 - update vbucketmap

> post
> /pools/default/buckets/<bucket_name>/updateVbucketMap

ok: 202

error:

410: update failure


Tap Source extension:
---------------------

We need to extend the current TapDumpSource to allow dumping data from vbucket list
new option: --vbucket-list, string, default is none

    get_tap_conn():
       if self.vbucket_list:
          tap_opts[coucbaseConstants.TAP_FLAG_LIST_VBUCKETS] = ''

Tap Sink design:
----------------

TapSink will subclass from pump_mc.CBSink with the following overwriting functions:

- check_base()

>   //allow destrination vbucket state
> to be replica and pending


 - fineconn(mconns, vbucket_id)

        rc, conn = super(TapSink, self).findconn(mconns, vbucket_id)
        if rc != 0:
            return rc, None
        tap_opts = {memcacheConstants.TAP_FLAG_SUPPORT_ACK: ''}
        conn.tap_fix_flag_byteorder = version.split(".") >= ["2", "0", "0"]
        if self.tap_conn.tap_fix_flag_byteorder:
          tap_opts[memcacheConstants.TAP_FLAG_TAP_FIX_FLAG_BYTEORDER] = ''
        ext, val = TapSink.encode_tap_connect_opts(tap_opts)
        conn._sendCmd(memcacheConstants.CMD_TAP_CONNECT,
                     self.tap_name, val, 0, ext)
        return rv, conn