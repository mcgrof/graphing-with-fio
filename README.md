# Architectural review of Graphing results with fio

fio lets you tests IO performance. Visualizing results is possible through
several means. This document provides an architectural review of graphing
data with fio through the known different means possible. The goal of this
document to enable storage software engineers familiarize themselves more
quickly with exactly what options are available, *exactly* how they work,
differences between the options available and enable easier nose diving into
the respective fio code. This document was also designed to clarify *why*
the author recommends that development and focus of `fiovisualizer` be phased
out and focus put into extending gfio for further needs.

After review with the community, it may be possible to incorporate some of
this documentation into fio. Other than further review with the community,
this documentation was also designed with markup language so further
style changes would be required for incorporation into fio upstream.

The graphing tools evaluated and covered in this document are:

  * fiovisualizer
  * gfio
  * fio_generate_plots

# Architectural review of data plotted by fiovisualizer

## About fiovisualizer

[fiovisualizer](https://github.com/intel/fiovisualizer) was started by Andrey
Kudryavtsev at Intel since it was *believed* that no graphical solution existed
upon project inception. Although this may *perhaps* be accurate in terms of
project inception, in so far as the public releases are concerned gfio dates
back *way* before fiovisualizer's inception.

fiovisualizer was released publically on January  5,  2015.
gfio          was released publically on February 24, 2012.

gfio however is not well documented, in fact there seems to be no documentation
about it to this day except how to enable it at build time via:

``` bash
./configure --enable-gfio
```

The only other documentation which exists is the
[gfio developer TODO](https://fio.readthedocs.io/en/latest/fio_doc.html#gfio-todo).

It may be possible that gfio was simply overlooked due to lack of any
documentation when considering development of fiovisualizer. Perhaps this
document may now serve as an initial form of architectural documentation for
it.

## Developer language choice for fiovisualizer

fiovisualizer is designed in Python, it relies on pyqtgraph.Qt. It splits
implementation between the GUI implementation on the `fio_visualizer.py` file
and calling the server on the `realtime_back.py` file.

## What fiovisualizer graphs

fiovisualizer relies on data points plotted by values returned by interval
status dumps.

It plots data for:

  * bw
  * clat
  * iops

## fiovisualizer use of terse ouput format v3

fiovisualizer relies on status dumps being issued by the backend server on
status intervals in terse version format 3, each line representing a status
dump per thread. fiovisualizer only parses certain fields per line:

```
| 6 | read_bandwidth |
| 7 | read_iops |
| 15 | read_clat_mean |
| 47 | write_bandwidth |
| 48 | write_iops | 23580 |
| 56 | write_clat_mean |
```

## fio's status dumps design review

To understand the efficiency of fiovisualizer's graphing methodology it is
important to first understand the design of how fio's status dumps are sent
to standard output, both if acting as a local server or if connecting to a
remote server as a client.

### fio status dump minimum time resolution and dealing with threads

fio's status dumps happen at a minimum status interval of every 100ms and
consists of *running average values*, not per *period values*, this fact
is clarified in the fio man page. By default the server prints results per line
per thread (numjobs). For instance:

``` bash
# egrep "^numjobs|^runtime" test.ini
numjobs=1
runtime=10

# fio --minimal --terse-version=3 --status-interval=1000ms --eta=never test.ini | wc -l
11
```

If we however edit the numjobs=2 we get more results on the same interval:

``` bash
# egrep "^numjobs|^runtime" test.ini
numjobs=2
runtime=10

# fio --minimal --terse-version=3 --status-interval=1000ms --eta=never test.ini | wc -l
22
```

This can be mitigated with `group_reporting=1`, so that results are shown
for cumulative for all threads.

The implications of *not* using `group_reporting=1` and having say the last
thread always report favorable conditions has not been empirically evaluated
yet, however this should be.

### Understanding fio status dump interval resolution

A check to see if we're within status_interval is done by
`check_for_running_stats()` and this check is done on each loop in the main
backend loop `run_threads()` after threads have been created and spawned to
run `thread_main()` via either `pthread_create()` (threads variable set in
the config file) or `fork()`. The call to `check_for_running_stats()` happens
within the `do_usleep(100000)` call, and so sleeps also after each call
for 100000 microseconds (100 milliseconds). The check to see if we're within
status_interval happens then every 100 milliseconds. This is a built-in
non-configurable value.

### fio status dump output

The actual status update print standard output is issued via:

``` C
	show_thread_status_terse(ts, rs, &output[__FIO_OUTPUT_TERSE]);
```

This is called on `__show_run_stats()`.

### fio status dump output to remote client

This arrangement was only recently fixed to work with a remote client/backend
server setup (as of 2018-09 via commit 1d1b65dc17c3 ("client: respect terse
output on client <--> backend relationship"), however this was recently
reverted and requires more work.  The relevant lines within
`__show_run_stats()` are:

``` C
void __show_running_run_stats(void)
{
	...
		if (is_backend) {
			fio_server_send_job_options(opt_lists[i], i);
			fio_server_send_ts(ts, rs);
			if (output_format & FIO_OUTPUT_TERSE)
				show_thread_status_terse(ts, rs, &output[__FIO_OUTPUT_TERSE]);
		} else {
			if (output_format & FIO_OUTPUT_TERSE)
				show_thread_status_terse(ts, rs, &output[__FIO_OUTPUT_TERSE]);
			if (output_format & FIO_OUTPUT_JSON) {
				struct json_object *tmp = show_thread_status_json(ts, rs, opt_lists[i]);
				json_array_add_value_object(array, tmp);
			}
			if (output_format & FIO_OUTPUT_NORMAL)
				show_thread_status_normal(ts, rs, &output[__FIO_OUTPUT_NORMAL]);
		}
	...
}
```

# OS support for fiovisualizer

fiovisualizer has been tested to work with Linux and OS X.

# Architectural review of data plotted by gfio

## About gfio

Stephen M. Cameron started gfio, and integrated as part of upstream fio.
Since then many others have given it love and extended it.

## Developer language choice for gfio

gfio is written in C and uses GTK.

## What gfio graphs

gfio shows performance metric data as an aggregate for all jobs.
On each graph window, it collects and graphs data for:

  * bw
  * iops

It therefore does not show on the main graph:

  * clat

In terms of graphing data it also allows you to visualize specific jobs
results. Its jobs tab therefore shows:

  * completion percentiles
  * latency buckets

## gfio graph setup

gfio sets up its graphs via `setup_graphs()`

gfio updates status updates per thread via its `struct client_ops`
`gfio_thread_status_op()` which is called per thread. This in turn
calls `gfio_display_ts()` just updates the final results tab however, displayed
when the full job completes.

Graphing occurs through two gfio `struct client_ops`, the `jobs_eta()` and
`eta()` callback. The graphing on the main window for gfio comes from the gfio
`struct client_ops` `eta()` callback, while client specific graphing is handled
through the `jobs_eta()` callback. The main window seems non-functional at the
time of this writing on *origin/master*. In fact, connecting to multiple clients
is also non-functional at the time of this writing.

Actual data graph data updates is performed through the helper
`graph_add_xy_data()` per eta update. On each graph window there are two graphs
a bw graphs and the iops graph.

## gfio client ops

The respective client ops:

``` C
struct client_ops gfio_client_ops = {
	...
        .eta                    = gfio_update_all_eta,
        .jobs_eta               = gfio_update_client_eta,
	...
};
```

## gfio - receiving eta updates

The jobs_eta() callback is called directly by handle_eta() when a client
receives a FIO_NET_CMD_ETA command.

client:
> FIO_NET_CMD_ETA --> handle_eta()
>   client->ops->jobs_eta()
>   client->ops->eta() (through fio_client_dec_jobs_eta())

## gfio - sending eta updates

The server sends the *FIO_NET_CMD_ETA* when it gets a request from
the client, that is, when the server recieves the *FIO_NET_CMD_SEND_ETA*
command.

server:
> handle_command(FIO_NET_CMD_SEND_ETA) --> handle_send_eta_cmd() (FIO_NET_CMD_ETA)

The `handle_send_eta_cmd()` function constructs the payload sent to the
client with the actual ETA data. It relies on `calc_thread_status()` to
extract the actual data. The final calculation for bandwidth and iops is done
with `calc_rate()`:

``` C
bool calc_thread_status(struct jobs_eta *je, int force)
{
	...
	calc_rate(unified_rw_rep, disp_time, io_bytes, disp_io_bytes, je->rate);
	calc_iops(unified_rw_rep, disp_time, io_iops, disp_io_iops, je->iops);
	...
}
```

The `calc_rate()` reveals to us how its actually the difference of data between
a prior run and current run which is stored for the ETA update, it keeps track
of the previously run values using a static variable. The data used for the
current and prior run are the summation of the values for all threads.  Data is
collected for each possible *Data Direction*, read/write/trim, for both
bandwidth and IOPS. The data is collected separately unless the option
`unified_rw_rep` is specified in which case a single summation of all
read/write/trim data is collected.

So for instance by default we'd collect the summation of:

 * bandwidth in bytes for read for all threads
 * bandwidth in bytes for write for all threads
 * bandwidth in bytes for trim for all threads

 * IOPS for read for all threads
 * IOPS for write for all threads
 * IOPS for trim for all threads

`calc_rate()` will compute the difference between the bandwidth in bytes for
each of these for all threads, and the difference is what is used to graph
and plot data. This is the aspect which makes ETA data per period.

The ETA updates then provide *per period* data evaluation for all threads as
an aggregate.

## gfio - client requesting for eta updates

The client requests for ETA data by sending the server with a
*FIO_NET_CMD_SEND_ETA* command, the client sends these requests in
`request_client_etas()`. Timeouts for not having received ETA updates from a
client are handled with `handle_cmd_timeout()`.

> client.c:
> request_client_etas() FIO_NET_CMD_SEND_ETA
> handle_cmd_timeout() checks if the command sent was FIO_NET_CMD_SEND_ETA

The `request_client_etas()` will issue a request for eta data on the time
interval specified by the client. After we request to issue an ETA with
`request_client_etas()` we wait for the client file descriptor to receive data
(POLLIN) with poll() for a minimum time between 100ms or the eta interval set
by the client. The loops is as follows:

``` C
int fio_handle_clients(struct client_ops *ops)
{
	...
	while (!exit_backend && nr_clients) {
		...
		do {
			struct timespec ts;
			int timeout;

			fio_gettime(&ts, NULL);
			if (eta_time_within_slack(mtime_since(&eta_ts, &ts))) {
				request_client_etas(ops);
				memcpy(&eta_ts, &ts, sizeof(ts));

				if (fio_check_clients_timed_out())
					break;
			}

			check_trigger_file();

			timeout = min(100u, ops->eta_msec);

			ret = poll(pfds, nr_clients, timeout);
			if (ret < 0) {
				if (errno == EINTR)
					continue;
				log_err("fio: poll clients: %s\n", strerror(errno));
				break;
			} else if (!ret)
				continue;
		} while (ret <= 0);
		...
	}
	...
}
```

So the resolution for the ETA interval is bounded by both the
`eta_time_within_slack()` check and then the response from the server which is
checked with `poll()` with a timeout of a response from the server set by
100 milliseconds. *However* upon initialization of fio, fio has a built-in
lower bound check for the minimum ETA interval to be no less than the
`DISK_UTIL_MSEC` which is currently set to 250 milliseconds.

fio_handle_clients() is called by fio.c on the main loop if it is not
a local backend. That is, if it is a client. Note that a local backend
sends updates to the stdout depending on the output format on the server
backend loop.

gfio calls fio_handle_clients() once connect_clicked() is called through its
`job_thread()` as a pthread, once the gfio client tries to connect to the server.
gfio allows one to graph data for different clients. It uses a main window
to represent overall progress for all clients. The "Main" windows is used
to represent ETA information for *all* clients. There are two callbacks
for a client then, one to manage updates for information from all clients, and
another callback to manage updates for only each specific client. The GUI
represents each client with a tab associated with the filename used for the
work opened, and keeps the "Main" data in its own first tab.

``` C
struct client_ops gfio_client_ops = {
	...
	.jobs_eta               = gfio_update_client_eta,
	.eta                    = gfio_update_all_eta,
	...
}
```

The `jobs_eta()` callback manages updates per client, meanwhile the `eta()`
callback manages the overal ETA for all clients. The eta() callback is called
indirectly by passing the function as an argument to a function,
`fio_client_dec_jobs_eta()`.

``` C
static void handle_eta(struct fio_client *client, struct fio_net_cmd *cmd)
{
	...
	if (client->ops->jobs_eta)
		client->ops->jobs_eta(client, je);
	...
	fio_client_dec_jobs_eta(eta, client->ops->eta);
}

static int fio_client_dec_jobs_eta(struct client_eta *eta, client_eta_op eta_fn)
{
	if (!--eta->pending) {
		eta_fn(&eta->eta);
		free(eta);
		return 0;
	}

	return 1;
}
```

The gfio jobs_eta() callback:

``` C
/*
 * Client specific ETA
 */
static void gfio_update_client_eta(struct fio_client *client, struct jobs_eta *je)
{
	...
	graph_add_xy_data(ge->graphs.iops_graph, ge->graphs.read_iops, je->elapsed_sec, je->iops[0], iops_str[0]);
	graph_add_xy_data(ge->graphs.iops_graph, ge->graphs.write_iops, je->elapsed_sec, je->iops[1], iops_str[1]);
	graph_add_xy_data(ge->graphs.iops_graph, ge->graphs.trim_iops, je->elapsed_sec, je->iops[2], iops_str[2]);
	graph_add_xy_data(ge->graphs.bandwidth_graph, ge->graphs.read_bw, je->elapsed_sec, je->rate[0], rate_str[0]);
	graph_add_xy_data(ge->graphs.bandwidth_graph, ge->graphs.write_bw, je->elapsed_sec, je->rate[1], rate_str[1]);
	graph_add_xy_data(ge->graphs.bandwidth_graph, ge->graphs.trim_bw, je->elapsed_sec, je->rate[2], rate_str[2]);
	...
}
```

The gfio eta() callback:

``` C
/*
 * Update ETA in main window for all clients
 */
static void gfio_update_all_eta(struct jobs_eta *je)
{
	...
	graph_add_xy_data(ui->graphs.iops_graph, ui->graphs.read_iops, je->elapsed_sec, je->iops[0], iops_str[0]);
	graph_add_xy_data(ui->graphs.iops_graph, ui->graphs.write_iops, je->elapsed_sec, je->iops[1], iops_str[1]);
	graph_add_xy_data(ui->graphs.iops_graph, ui->graphs.trim_iops, je->elapsed_sec, je->iops[2], iops_str[2]);
	graph_add_xy_data(ui->graphs.bandwidth_graph, ui->graphs.read_bw, je->elapsed_sec, je->rate[0], rate_str[0]);
	graph_add_xy_data(ui->graphs.bandwidth_graph, ui->graphs.write_bw, je->elapsed_sec, je->rate[1], rate_str[1]);
	graph_add_xy_data(ui->graphs.bandwidth_graph, ui->graphs.trim_bw, je->elapsed_sec, je->rate[2], rate_str[2]);
	...
}
```
# OS support for gfio

gfio has been tested to work with Linux and OS X.

# Architectural review of data plotted by fio_generate_plots

fio writes data to files if you enable the variable such as `write_bw_log`,
`write_lat_log`, `write_hist_log`, and `write_iops_log`.

`fio_generate_plots` is a shell script which runs gnuplot against these
sorts of files to plot data into a graph with gnuplot.

Enabling any of these options will create initial `struct io_log` references,
cached globally in their respective agg_io_log[description], each reference
with a respective filename reference. Upon each ETA update, in
`calc_thread_status()`, each respective `struct io_log` is updated via
add_agg_sample() calls.

At the end of the `fio_backend()` run, each `struct io_log` are flushed to disk
via `flush_log()` calls, each file is created with the respective file name
and data from each. By default if you have n jobs then n files will be created
per operation recorded.

iolog.c implements overal mangement of iolog, `flush_samples()` and
`flush_hist_samples()` write the data to the files.

`fio_generate_plots` is a small script, ~132 lines, it leverages gnuplot to
graph results per thread/fork (jobs in file) for each of the following:

  * I/O Latency - Time (msec)
  * I/O Operations Per Second - IOPS
  * I/O Submission Latency - Time (μsec)
  * I/O Completion Latency - Time (msec)
  * Time (msec) - Throughput (KB/s)

There are 8 primary colors used for graphing data per graph, each color is
used per thread/fork (jobs). If there are more than 8 threads/forks each
color is differentiated further by delimeters on the line. That said,
with more than 8 jobs results are difficult to comprehend. For this reason
it may also be sensible to consider using `group_reporting=1` when evaluating
graphs with `fio_generate_plots`. Without `group_reporting=1`, if there is
any significant event on the system which would for whatever reason allow for
one job to outperform others, an outlier, results can easily be skewed and may
difficult to analyze.

At least with the default options then `fio_generate_plots` would be suitable
for capturing outliers if more than one job is used, it however makes it
difficult to get a clear idea of overall performance evaluation unless
`group_reporting=1` is used.

`group_reporting=1` however does provide *per period* evaluation of performance.

# Comparing and contrasting fiovisualizer, gfio and fio_generate_plots

Below is a compare and contrast of offering of different attributes for
graphing to consider between all options available.

## Per thread graphing implications

Currently fiovisualizer relies on average data per thread, and plots all thread
data on one graph. If more than one job is used and `group_reporting=1` is not
used then a new line will be printed per job per status interval, so data
represented may be skewed. Using `group_reporting=1` will group data output
per line and avoid this issue. gfio does not have to deal with these issues
as its client deals with processing data on a callback, and on such callback
processes all thread data as needed.

## Parsing Vs direct client callback approach

fiovisualizer also parses output data issued by the server. Parsing data incurs
an overhead. fio provides better mechanisms to implement clients without the
need to incur parsing. If more *real-time* graphing is desirable, implementing
a client with proper callbacks would be more efficient, as gfio does.

If the same functionality is still desired, fiovisualzier could be
re-implemented and integrated into fio as a new proper client and deal with its
processing of data through the `struct client_ops` `thread_status()` callback,
thereby skipping parsing.

## Data source selection and implications

The data source used by fiovisualizer is important to analyze further.
fiovisualizer is also plotting average thread data, not per period data. This
is because it is working with thread status updates and thread status updates
are not per period, they provide overall average data results.

The visual implications of plotting aggregate average status updates per
thread and not per period should be considered when analyzing fiovisualizer
graph results. For instance, if it is desired to display large differences in
performance per job, for example to catch outliers per job, it should be aiming
to plot per period performance metrics, not average results. Average results
would hide outlier performance gaps. Additionally since average data is used,
one should realize any anomalies observed on the graph may actually be more
radical than what is revealed on the graph.

If one goal is to really analyze per job metrics, graphing complete performance
per period individually would provide a better visualization of results. Using
eta updates provides this data source, and its how gfio implements its graphical
client.

Since gfio relies on per period data its visual representation of performance
results reveal more accurately what is occurring at the time of each plot.

Usage of fio_generate_plots relies on per period data, however, since data per
job is split up, and there can be outliers, results can easily be skewed and
therefore difficult to read unless `group_reporting=1` is used.

## Real time analysis

`fiovisualizer` claims to provide realtime graphing, however `gfio` provides
only 150 milliseconds better resolution than gfio. Anything below 100
milliseconds for status intervals currently reveals more data but the accuracy
on timing below may not be what be expected and requires further investigation.
Additionally there is a time penalty to consider on `fiovisualizer` as it parses
data prior to plotting, this impact should be considered, however it is expected
to be minor.

`fiovisualizer` relies on status interval updates. Although the interval is
configurable, the check for such interval *should* only happen every
statically built-in `do_usleep(100000)` call, so every 100000 microseconds
(100 milliseconds). In practice though we currently allow for status interval
values less than 100 milliseconds, and 10ms resolutions seems to provide more
data as expected over the same period of time, but anything beyond 10ms does
not provide more data:

``` bash
# egrep "^numjobs|^runtime" test.ini
numjobs=2
runtime=10

# fio --minimal --terse-version=3 --status-interval=100ms --eta=never test.ini | wc -l
100

# fio --minimal --terse-version=3 --status-interval=10ms --eta=never test.ini | wc -l
991

# fio --minimal --terse-version=3 --status-interval=1ms --eta=never test.ini | wc -l
990
```

The discrepancy between these needs to be analyzed and a lower limit properly
defined and documented in fio.

gfio relies on ETA interval updates. The resolution of the ETA interval is
bounded by the ETA interval itself and the responses from the server and a
minimum timeout of 100 milliseconds response from the server for the update.
The built-in lower boundary to have to be higher than 250 milliseconds is
what currently sets the limit to the minimum ETA interval.

Likewise `fio_generate_plots` relies on the same ETA interval specified, and
shares the same resolution as with gfio.

## Final questions

This all begs a few questions which would be best discussed with the community:

  * Which is truly a better data source for graphing and why? Perhaps this is
    a function of what you want to look for and evaluate performance for.
  * At what time interval are we graphing in *realtime*?
  * Is the short possible gain observed today between status interval updates
    Vs ETA interval updates really effective for graphing and providing
    additional assistence?
  * Can and should fio's own lower limit of higher than 250 milliseconds be
    lifted? Why?
