## Initial Profiling of Cdatools Export
I could see that the Go library's memory consumption was concerning Dave, so I decided to work on profiling the code a bit before getting back to cat3 templates.

The code is actually pretty darn fast, it's just that mainly go-bindata is generating a lot of garbage for us. Here are my findings:

## Go profiling
After making a Benchmark, Go provides some tools to create and view cpu and mem profiles based off that Benchmark, or many Benchmarks for that matter. It also provides a handy feature to view them in svg format so that they can be viewed in a browser. This provides a call stack/tree/graph of the GenerateCat1 function benchmark, which makes it very to easy to see problem areas. The bigger the box and font containing a function call, the bigger of a potential problem that function is.
### CPU
Knowing the information above, I first decided to check out CPU and see where most of the thinking was going on, as that usually gives a road map to the problem areas. Sure enough, looking at the initial cpu profile in `prof-initial-cpu.svg`, it shows that 53% of time is spent in GC. Wat the wat!?

![](/static/poolshark.jpg)

>Honestly, considering that our code is mainly a bunch of if-statement-business-logic and parsing templates, I think the majority of our CPU time will, relatively, be GC. But this at least confirms this suspicion.

### RAM

Knowing that the CPU is working overtime to keep our memory consumption down, we turn to a memory profile to see if we can make its job a little easier. So let's open up the mem profile at `prof-initial-mem.svg`!

> At first, this is comedic in nature, as one seems to have happened upon a strange mock up of The Captain's Deck of The Starship Enterprise.

But upon scrolling further to the right a ways, all one sees is a modest box informing the reader that 55% of the memory being allocated in our application is inside a go-bindata function, which is calling a function I've never even heard of before.

![](/static/watowl.png)

In order to answer my inner owl, I used `go tool pprof`'s ability to list a particular function's information in order to get a better understanding as to why this was happening. Here's the dump of bindataRead:
```
Total: 350.60MB
ROUTINE ======================== github.com/projectcypress/cdatools/exporter.bindataRead in /Users/mbianchi/src/github.com/projectcypress/cdatools/exporter/templates.go
       1MB   214.43MB (flat, cum) 61.16% of Total
         .          .    130:	"strings"
         .          .    131:	"time"
         .          .    132:)
         .          .    133:
         .          .    134:func bindataRead(data []byte, name string) ([]byte, error) {
  512.05kB   197.89MB    135:	gz, err := gzip.NewReader(bytes.NewBuffer(data))
         .          .    136:	if err != nil {
         .          .    137:		return nil, fmt.Errorf("Read %q: %v", name, err)
         .          .    138:	}
         .          .    139:
  512.05kB   512.05kB    140:	var buf bytes.Buffer
         .    16.04MB    141:	_, err = io.Copy(&buf, gz)
         .          .    142:	clErr := gz.Close()
         .          .    143:
         .          .    144:	if err != nil {
         .          .    145:		return nil, fmt.Errorf("Read %q: %v", name, err)
         .          .    146:	}
```

And then the dump of compress/flate.NewReader just out of curiosity and the fact that it was the real culprit underneath gzip.NewReader.
```
Total: 350.60MB
ROUTINE ======================== compress/flate.NewReader in /usr/local/Cellar/go/1.8/libexec/src/compress/flate/inflate.go
  194.39MB   194.39MB (flat, cum) 55.44% of Total
         .          .    782://
         .          .    783:// The ReadCloser returned by NewReader also implements Resetter.
         .          .    784:func NewReader(r io.Reader) io.ReadCloser {
         .          .    785:	fixedHuffmanDecoderInit()
         .          .    786:
   24.61MB    24.61MB    787:	var f decompressor
         .          .    788:	f.r = makeReader(r)
   14.04MB    14.04MB    789:	f.bits = new([maxNumLit + maxNumDist]int)
       1MB        1MB    790:	f.codebits = new([numCodes]int)
         .          .    791:	f.step = (*decompressor).nextBlock
  154.74MB   154.74MB    792:	f.dict.init(maxMatchOffset, nil)
         .          .    793:	return &f
         .          .    794:}
         .          .    795:
         .          .    796:// NewReaderDict is like NewReader but initializes the reader
         .          .    797:// with a preset dictionary. The returned Reader behaves as if
```

## To Sum It Up
When looking through this code, it seems like the tool `go-bindata` reads the template files and gzips them before turning them into byte slices in the `templates.go` file when we run `make`. We really don't need this, since we don't have extreme concerns with our binary size and the exporter.test only grew about .1MB. What we are concerned with is that this function takes a lot of RAM.
Upon reading the `go-bindata` README, my assumptions on gzipping are correct. Luckily there is a -nocompress option, which I added to the Makefile so I could use another go tool, `benchcmp`, in order to see if we got any performance boosts off a change in a cli command. Here are the results:
```
benchmark                   old ns/op     new ns/op     delta
BenchmarkGenerateCat1-4     10652038      8517520       -20.04%

benchmark                   old allocs     new allocs     delta
BenchmarkGenerateCat1-4     16243          15632          -3.76%

benchmark                   old bytes     new bytes     delta
BenchmarkGenerateCat1-4     3977365       1481435       -62.75%
```

>Sweet. One change takes us from 4MB/op to 1.5MB/op!
So less ns are going by per op and 2.5M bytes less per op. SGTM.

Let's look at the new profiles (mem first) named `prof-no-compress-<cpu/mem>.svg`. When looking at the graph, it seems like -no-compress did its job. If we follow the large arrow that indicates the largest mem allocations, it goes down our actual computation path into `EntryInfosForPatient`.

## But why stop here? 
The new culprit for memory problems is now in the `regexp` package under `GetEntriesForOids`. This comes under the `MustCompile` function, which has to compile the string and make a regex. This regex has no dynamic information in it, so recompiling it on every function call is pretty wasteful. Let's see what happens when we move it out:
```
benchmark                   old ns/op     new ns/op     delta
BenchmarkGenerateCat1-4     8517520       8121824       -4.65%

benchmark                   old allocs     new allocs     delta
BenchmarkGenerateCat1-4     15632          15392          -1.54%

benchmark                   old bytes     new bytes     delta
BenchmarkGenerateCat1-4     1481435       1171029       -20.95%
```

Yay! We managed to get rid of all 20% of those memory allocations by keeping just one reference to the regex. Time for a beer. Or sleep... definitely beer.

## Where to go from here
Now, looking at the final two profiles from this initial investigation, the graphs look a lot less concerning with most of the memory allocations and compute time spent in the "real work" path.

The cpu profile points straight down the template parsing code into GC, as anticipated, and the memory profile has a nice little spike going down the template code, as expected. I also anticipated the spin offs in the memory profile to the functions that do a lot with the "more liberally used than preferred" maps in our system.

There also seems to be an opportunity to get 50MBs worth of memory back from our system if we find a way to stick with byte slices or strings and avoid having so many byte slice to string transformations (but that might be the nature of the beast).

I do have confidence that we can clean up and reduce the use of these maps but this will eventually be a necessary evil, since the ruby code gets to use mongo to store and query this data while the Go code needs to hold it all in memory.

We could also possibly reduce the bytes-strings dilemma by figuring out where we're swapping from bytes to strings so much and keeping the data in one world longer in order to shrink memory consumption just a bit more.

But, for now, here's the benchcmp between the initial code when I started and the two changes I made to increase performance:
```
benchmark                   old ns/op     new ns/op     delta
BenchmarkGenerateCat1-4     10652038      8121824       -23.75%

benchmark                   old allocs     new allocs     delta
BenchmarkGenerateCat1-4     16243          15392          -5.24%

benchmark                   old bytes     new bytes     delta
BenchmarkGenerateCat1-4     3977365       1171029       -70.56%
```
