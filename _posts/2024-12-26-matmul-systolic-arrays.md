---
layout: post
title: "Project Deep Dive: Matrix Multiplication with Systolic Arrays"
date: 2024-12-26 15:00:00
categories: engineering vlsi code projects
description: "A writeup about a course project (EE 477 @ USC, Fall 2024) about Systolic Array Matrix Multiplication"
tags: [usc, engineering, vlsi, code, projects]
---

# Setting the Stage

Three students walk into the collaborative study space at
[Leavey Library](https://today.usc.edu/5-things-you-need-to-know-about-leavey-library/),
packing two months of instruction on MOS transistors, a project description
detailing several dozen hours of work, and some (convenient, if perhaps
economically ill-advised) starbucks coffees. None of us have prior exposure to
the VLSI industry. One of those students is me ☺

(This was our first real foray into VLSI design. While I've performed
schematic/layout tasks as part of my job at Emerson, the skills I developed for
analog PCB design largely did not apply to digital MOS design tasks at this
scale, outside of some keybind similarities between Cadence Virtuoso and
Allegro System Capture. Ergo, all three of us started with fresh slates.)

For the first two months of this course, we completed contributory design
activities without knowing the project spec (which changes on a semesterly
basis). Our prior labs saw us design a set of
[Complementary MOS](https://en.wikipedia.org/wiki/CMOS)
(CMOS) logic gates, out of which we built
[Ripple-Carry](https://www.circuitstoday.com/ripple-carry-adder) and
[Carry-Bypass](https://en.wikipedia.org/wiki/Carry-skip_adder) adders using a
hierarchy of standard cells. Finally, after our first exam in November
the project spec dropped:

# Project Description

**Our task:** Design a systolic array matrix mulitplier for 4x4 matrices,
consisting of 4-bit unsigned integers.

A refresher on matrix multiplication (since I always seem to need it too):

1. Take the _i_-th row of the first matrix
2. Multiply all elements against their counterparts in the _j_-th column of the
   second matrix
3. Sum all the products together. That's the value of element $$c_{i, j}$$

$$\textbf{AB = C}$$

$$
\begin{bmatrix}
a_{1,1} & a_{1,2} & a_{1,3} & a_{1,4} \\
a_{2,1} & a_{2,2} & a_{2,3} & a_{2,4} \\
a_{3,1} & a_{3,2} & a_{3,3} & a_{3,4} \\
a_{4,1} & a_{4,2} & a_{4,3} & a_{4,4}
\end{bmatrix}
\begin{bmatrix}
b_{1,1} & b_{1,2} & b_{1,3} & b_{1,4} \\
b_{2,1} & b_{2,2} & b_{2,3} & b_{2,4} \\
b_{3,1} & b_{3,2} & b_{3,3} & b_{3,4} \\
b_{4,1} & b_{4,2} & b_{4,3} & b_{4,4}
\end{bmatrix}
=\begin{bmatrix}
c_{1,1} & c_{1,2} & c_{1,3} & c_{1,4} \\
c_{2,1} & c_{2,2} & c_{2,3} & c_{2,4} \\
c_{3,1} & c_{3,2} & c_{3,3} & c_{3,4} \\
c_{4,1} & c_{4,2} & c_{4,3} & c_{4,4}
\end{bmatrix}
$$

where:

$$c_{i,j} = \sum_{k=1}^{4} a_{i,k} \cdot b_{k,j}$$

(for example:)

$$c_{1, 2} = a_{1, 1}b_{1, 2} + a_{1, 2}b_{2, 2} + a_{1, 3}b_{3, 2} + a_{1, 4}b_{4, 2}$$

This is a much more involved project than our prior labs, and we haven't even
gotten to the central question of the project:

## What are systolic arrays, and how do they work?

[**Systolic arrays**](https://www.telesens.co/2018/07/30/systolic-architectures/)
are parallel hardware architectures constructed of arrays of processing
elements. Each processing element performs part of a task and stores part of
the result. This is useful for matrix multiplication, where there are many
separable subtasks (i.e., elements to compute) with many intermediates (i.e.,
partial sums).

Data flows between adjacent processing elements. This:

- Keeps wires short (improving interconnect timings, which are otherwise the
  [main timing bottleneck](http://async.org.uk/noc2006/pdf/Eby-Friedman.pdf))
- Enables pipelining data between elements (good for matmul!)
- Organizes I/O so that it only appears at the edges of the array

![Systolic Array Matrix Multiplication Figure](/assets/img/samm/Systolic%20Array%204x4.drawio.jpg){:width="100%"}

The above figure details how data flows through the systolic array for matrix
multiplication:

1. The multiplicand matrix $$\textbf{A}$$ streams left-to-right through the rows
   of processing elements, along the indicated interconnects. (Each row of the
   matrix goes through the corresponding row of processing elements.)
2. The multiplier matrix $$\textbf{B}$$ streams from top to bottom through the
   columns of processing elements. (Each column of the matrix goes through the
   corresponding column.)
3. The product matrix is iteratively produced inside the processing element
   (where the position of the processing element matches the position of the
   matrix element which is stored). Id est, the element in the fourth row,
   third column stores $$c_{4, 3}$$.

Both input matrices _pipeline_ (visit, and then leave) through the
processing elements in the indicated directions.

- Each four-bit element of the matrix spends one clock cycle in a processing
  element before moving to the next.
- Therefore, a series of four numbers:
  - Takes four clock cycles to pass through a single processing element, and
  - Takes seven clock cycles to pass through a row or column completely.

The $$\textbf{A}$$ and $$\textbf{B}$$ matrix pass through the systolic array
_simultaneously_ and in a "diagonal" fashion, so that the necessary elements
needed to produce partial sums are present in their correct processing elements
at the correct times. When the individual product elements are ready, they are
transmitted to the output pins at the top of the array.

This isn't a super intuitive explanation, so instead I'll link to this GIF:

<div style="width:100%;height:0;padding-bottom:89%;position:relative;">
    <iframe src="https://giphy.com/embed/8vZUh9LrkN4llUUxaN" 
        width="100%" 
        height="100%" 
        style="position:absolute" 
        frameBorder="0" 
        class="giphy-embed" 
        allowFullScreen
    ></iframe>
</div>
<p>
    <a href="https://giphy.com/gifs/systolic-matrix-multiplication-8vZUh9LrkN4llUUxaN">
        via GIPHY
    </a>
</p>

The GIF uses a different indexing system than I do, so I'll limit my analysis
to the following observations:

1. The rows/columns sent through the matrix start their journeys one cycle
   apart. Row $$i$$ of matrix $$\textbf{A}$$ starts at cycle $$i$$, which is exactly
   when column $$i$$ starts as well.
2. Each processing element at position $$(i, j)$$ will see the $$i$$-th row of
   $$\textbf{A}$$ and the $$j$$-th column of $$\textbf{B}$$.
3. These are the correct numbers to produce $$c_{i, j}$$, which is available
   _immediately after_ the required row/column finishes passing through the
   processing element (at the top of the next clock cycle). Details for exactly
   how this happens are discussed later.

This enables a space-efficient, high-frequency, and relatively simple matrix
multiplication implementation, which matches well with how the project was
setup:

## Goal: Optimizing for Power, Area, and Delay (PAD)

This project was setup as a course contest, with PAD minimization being the
sole judging criteria. (Also important: the project has to work! This is an
important point which will be discussed later.)

- **Power** is judged by how many microwatts the device consumed while
  producing a matrix product
- **Area** is judged by how many square microns the device footprint requires.
  We were additionally told that our device should be as square as possible, as
  this criterion was tweaked to calculate area as the square of the longest
  dimension.
- **Delay** is judged by how long the device takes to calculate the product.
  For simplicity, this criterion is reduced to be a measurement of the clock
  cycle period.

> (PAD is also known as PPA: Power, Performance, and Area.)

This project was split into three phases:

- **Phase 1**: Design of multiply-accumulate hardware (to iteratively acquire the
  product elements)
- **Phase 2**: Design of the processing element and a vector multiplier
  (a single column of processing elements)
- **Phase 3**: Systolic Array Design (using the vector multiplier
  designed earlier)

We chose to largely forgo aggressive design optimizations (such as transistor
sizing, non-CMOS logics, and routing variants to minimize parasitics) so
that we could balance this project with the collection of other coincident
labs, exams, and assignments. This meant we largely stayed within a
hierarchical standard-cell design workflow, utilizing CMOS and
[Transmission-Gate Logic](https://en.wikipedia.org/wiki/Transmission_gate)
(TGL).

For our course we used GPDK045 with Cadence Virtuoso to implement our designs.

# Phase 1: MAC Building blocks

There are three components required for multiply-accumulate hardware on 4-bit
unsigned integers:

1. A four-bit multiplier (produces 8-bit products)
2. A 10-bit adder, accepting the product as one operand and an intermediate sum
   as the other
3. A 10-bit register, connected with the adder in a feedback loop

Diagram:

![MAC](/assets/img/samm/MAC.drawio.jpg){:width="100%"}

We chose to use an array multiplier and a ripple-carry adder (RCA) because our
bit counts were too low to justify implementing more advanced adder/multiplier
designs. (Discussions with course TAs and the professor advised us that they
would not have been faster.) We used a TGL master-slave D flip flop design for
our register.

## Division of Work (and the first trap laid)

Now, one may think that three team members and three components produces a very
obvious work division scheme of one component per member.

This may have worked if each member acted as a 'lead' for development of that
component and pulled in other team members as needed. However, we did not yet
have our collaboration routines grooved, and largely chose to implement our
respective components individually. This was very bad for scheduling, as the
one who claimed responsibility for the multiplier found their design task
extremely daunting. (Keep in mind, we had a ripple-carry adder design from
the previous labs, and in comparison to a register design, the multiplier
design is far more involved.)

By the time we learnt our lesson, our multiplier man had spent several days of
staring at angry
[Design Rule Check](https://en.wikipedia.org/wiki/Design_rule_checking) (DRC)
screens with hundreds of errors. Not all was doom and gloom, however.

## Epiphany I: Surpassing hierarchical standard cell design

Several improvements were enabled in the slack time resulting from the
multiplier effort.

The RCA adder was redesigned with smaller logic gates and a placement scheme
that vertically stacked pairs of mirrored full adders such that their
power/ground rails were abutted.

The register underwent a full-custom redesign complete with individual
transistor sizing that resulted in a 50ps improvement in propagation delay
(from 120 down to 70) and a ~60% reduction in size.

The resulting multiplier design was extremely aggressive in its optimization
strategies. Our engineer had completed a first pass design that passed DRC, LVS
(Layout Versus Schematic), and the functional verification test suite provided
to us as part of project spec. His second design aggressively reduced the
spacing between transistors and stripped the pads on the cell templates for
some standard cell variants, allowing their power/ground rails to overlap
seamlessly. Both of these changes invalidated some DRC and LVS rules, but
produced an effectively full-custom multiplier design that passed both suites
of checks.

The resulting register and adder designs align nicely. However, the multiplier
design is very long, as we elected to reserve one 'row' of cell template per
bit of input operand. Placing the output pins at the bottom of the multiplier
and rotating the multiplier by $$90^\circ$$ ameliorates the issue.

![MAC Layout](/assets/img/samm/MAC%20Layout.jpg){:width="100%"}

Remarks:

- We had not yet 'extracted' the input pins from the multiplier as we
  were unsure where they would be routed in future phases of the project.
- The register contains a reset signal to initialize the MAC register contents
  between multiplications and on power-on.

# Phase 2: Systolic Array Processing Elements

The processing element consists of a few resistors to provide data storage for
the input data lines and a multiplexer to selectively expose the stored product
(when it is time). The multiplexer has a control signal which will pass along
processing elements in a row and must also be registered.

![Matrix Multiplication Processing Element](/assets/img/samm/matrix%20multiplication%20processing%20element.jpg){:width="100%"}

Within the diagram:

- The $$\textbf{A}$$ input lines start at left, feeding to the multiplier
  and tracing across the processing element, getting registered, then appearing
  at the right side.
- The $$\textbf{B}$$ input lines start at top, feeding to the multiplier,
  then getting registered and appearing on the bottom side.
- The `sel` input lines trace left-to-right, getting registered before getting
  fed to the multiplexer.
- The `sum_out` output pins at top present data controlled by the
  `sel_reg`-controlled multiplexer. This is either the internal MAC product
  (if `sel_reg` is 0) or a passed-along version of the `sum_out` data from a
  processing element below this one. (There is a set of `sum_in` pins present
  on the bottom side.)

The layout of the design largely matches the diagram. There is a
not-insignificant amount of decision-making needed to determine component
placement of the three extra registers within the processing element (all of
which use the same design as in MAC). However, there is a more pressing
question:

## How should we route signals through the processing element?

This question is twofold:

1. How should the wires cross between components to minimize delay?
2. Where should the wires be routed?

These concerns are made nontrivial by our lack of knowledge about Virtuoso's
built-in automated routing routines (which we were not disallowed from using,
but were not instructed on how to use). We therefore had to complete routing
largely by hand.

We quickly adopted systematic approaches to wiring decisions at this level,
because we observed that interconnect delays started to get somewhat unwieldy
at the end of phase one within the MAC.

One approach that improved our results was a practice of reusing as much of
lower-level routing as possible: Virtuoso only seems to recognize net
connections at higher levels to lower levels in a hierarchical design by
routing them through the pins. This can be bypassed by laying the wire on the
same net within the lower level design (using the same metals). Furthermore, if
one can identify points along such internal nets that are closest to each
other, this was observed to improve interconnect delays, if only minutely. We
did not use this strategy to its highest extent but it did inform our routing
decisions.

We didn't find strategies for reducing layout routing difficulty. (For the
processing element, this was much harder than the RCA and register design
tasks, if not as hard as the multiplier.) One thing that did make our lives
easier was configuring Virtuoso to show layout pin labels (and labelling our
pins manually). However, this still meant we had dozens of connections to make,
accessing nets which are largely without labels (since they exist on lower
levels of the hierarchy).

What I am trying to say with all this writing: routing for the processing
element was a hard task. We foresaw this to be even worse with the vector and
matrix multipliers. We wanted to make our lives easier, so we did this in
response:

## Epiphany II: Aligned I/O tracks for easier downstream layout tasks

Our strategy had three prongs:

1. Input and output pins must be placed such that they are aligned.
2. Horizontal nets and vertical nets must use a dedicated, different metal
   layer to the maximum extent possible.
3. Multi-bit signals must be laid out such that they use multiple adjacent and
   tightly-arranged routing channels.

This produced a very intelligible and somewhat-advantaged routing scheme.
Multi-bit channels (particularly `sum_out`) always fed to internal pins which
themselves were aligned (i.e., had the same x or y coordinate). Placing a
group of routing tracks centred on this alignment coordinate made it very
straightforward to interface with multi-bit signals.

Our strategy of dedicated metal layers also meant we were able to keep our
processing element design (_as well as both multipliers thereafter!_) to only
four metal layers, which helps reduce interconnect delay.

Finally, aligning the I/O pins meant that when we string multiple processing
elements together, it is trivial to route their connections. (All one needs
to do is place straight, short metal wires to connect the vast majority of
signals. The only two that need some work are clock and reset pins, which will
be discussed later.)

In connection with a decision to place the registers in the open space above
and below the MAC RCA/register, the end result is a very square layout:

![Processing Element Layout 1](/assets/img/samm/processing_element.jpg){:width="100%"}

The pin-out largely matches the diagram. Details:

- Component placement:
  - The 4-bit registers for $$\textbf{A}$$ and $$\textbf{B}$$ are placed below
    and above the MAC adder/register circuitry, respectively. They are arranged
    such that the individual flip flops are aligned horizontally (instead of
    vertically) to make efficient use out of the empty space.
  - The 1-bit register for the `sel` signal is placed in the lower right
    corner.
- Left side:
  - The clock pin is placed at the middle of the left side
  - The $$\textbf{A}$$ element input pins are placed well below `clk`
  - The `sel` pin is placed at the bottom of the left side, below the
    $$\textbf{A}$$ pins.
- Right side:
  - The registered $$\textbf{A}$$ output pins are placed on the right side,
    aligned with their counterparts on the left.
  - The `sel_reg` pin is likewise placed on the right
- Top side:
  - The $$\textbf{B}$$ input pins are placed above the adder/register hardware
    spaced evenly along the top side. There is plenty of space between these
    pins (which could potentially be reduced in more optimal designs).
  - The `sum_out` pins are placed on the top side, tightly clustered towards
    the right corner. This was done so they can all be routed to/from the
    multiplexer, which was placed to the right of the MAC.
- Bottom side:
  - The registered $$\textbf{B}$$ output pins are placed on the bottom side,
    aligned with their inputs up top.
  - The `sum_in` input pins are placed in a tight cluster on the bottom right
    corner, aligned with their `sum_out` counterparts.
  - The reset pin is placed on the bottom side between two registered
    $$\textbf{B}$$ output pins, directly aligned over the internal reset pins
    for the $$\textbf{A}$$ and $\textbf{B}$$ registers.

These choices simplify future layouts by making it easier to bridge signals
between processing elements and simplifying the paths that propagating signals
need to take (straight lines, minimal corners and vias).

# Phase 3: Vector and Matrix Multipliers

We leveraged the processing element to construct:

- A vector multiplier (by placing 4 processing elements in a column), and
- A matrix multiplier (by placing 4 vector multipliers in a row).

Vector multiplier layout:

![Vector multiplier layout](/assets/img/samm/vector_layout_multiview.jpg){:width="100%"}

Matrix multiplier layout:

![Matrix multiplier layout](/assets/img/samm/matrixmultiplier.jpg){:width="100%"}

These multipliers use the internal multiplexers to expose their results.
When the internal MAC sum finishes accumulating products:

1. At that cycle, the processing element receives `sel=0`.
2. This signal is registered, and at the next clock cycle appears as
   `sel_reg=0`.
3. When `sel_reg=0`, the multiplexer exposes the MAC sum on the `sum_out` pins
   for that column of processing elements.

This produces a "diamond" of results: one cycle after the last inputs of the
first column and first row are sent, the MAC sum of the top left processing
element is displayed on `sum_out`. In next cycle, the processing elements to the
bottom and right will display their MAC sums on their respective `sum_out`
lines. This continues until, six clock cycles after the first sum is produced,
the last MAC sum (from the bottom right processing element) is displayed on
the `sum_out` pins of the last column.

Now that we've constructed a prototype for our final project, there remains the
need to test it. Our course's initial workflow of defining stimuli within
[Virtuoso's built-in stimuli editor](https://community.cadence.com/cadence_blogs_8/b/cic/posts/virtuosity-creating-and-previewing-stimuli)
becomes too unwieldy for designs of this scale. (We need four stimuli signals
per four-bit signal, and now we have eight of those!) Luckily, Virtuoso
supports specifying multiple stimuli signals via
[digital vector files](https://eda.engineering.wustl.edu/wiki/index.php/Tutorials:Cadence:AdvancedTopics),
but the process for writing them is still tedious. What if we want to use  
multiple test cases? Writing multiple 4x4 matrices in binary notation by hand
is not how my team wanted to spend our afternoons. I made a quick solution to
resolve this bind:

## Epiphany III: Automating Vector Files for Functional Verification

Our course's fourth lab (assigned during the project) introduced python as a
means of automation for things like data parsing, generation, analysis, etc. I
saw an easy opportunity to apply this to writing vector files containing our
multi-bit signals.

```python
def _hex(i):
	return hex(i)[2].upper()

def make_case(a, b):
	"""Makes a systolic array case

	Args:
    	a (List[List[int]]): A 4x4 multiplicand matrix (represented as a list of lists)
    	b (List[List[int]]): A 4x4 multiplier matrix (represented as a list of lists)

	Returns:
    	List[List[int]]: A systolic array case (represented as a list of lists) that performs matmul(a, b)
	"""
	return f"""
;autogenerated systolic array case for matmul(a, b)
;------------------------------------------------------------------------------
;{_hex(a[0][0])} {_hex(a[0][1])} {_hex(a[0][2])} {_hex(a[0][3])} | {_hex(b[0][0])} {_hex(b[0][1])} {_hex(b[0][2])} {_hex(b[0][3])}
;{_hex(a[1][0])} {_hex(a[1][1])} {_hex(a[1][2])} {_hex(a[1][3])} | {_hex(b[1][0])} {_hex(b[1][1])} {_hex(b[1][2])} {_hex(b[1][3])}
;{_hex(a[2][0])} {_hex(a[2][1])} {_hex(a[2][2])} {_hex(a[2][3])} | {_hex(b[2][0])} {_hex(b[2][1])} {_hex(b[2][2])} {_hex(b[2][3])}
;{_hex(a[3][0])} {_hex(a[3][1])} {_hex(a[3][2])} {_hex(a[3][3])} | {_hex(b[3][0])} {_hex(b[3][1])} {_hex(b[3][2])} {_hex(b[3][3])}
;------------------------------------------------------------------------------
;ape4<[3:0]>,ape3<[3:0]>,ape2<[3:0]>,ape1<[3:0]>,bmvm1<[3:0]>,bmvm2<[3:0]>,bmvm3<[3:0]>,bmvm4<[3:0]>,sel[4:1],reset
0 0 0 0 0 0 0 0 0000 0 ; reset
{_hex(a[0][0])} 0 0 0 {_hex(b[0][0])} 0 0 0 1111 1 ; load
{_hex(a[0][1])} {_hex(a[1][0])} 0 0 {_hex(b[1][0])} {_hex(b[0][1])} 0 0 1111 1
{_hex(a[0][2])} {_hex(a[1][1])} {_hex(a[2][0])} 0 {_hex(b[2][0])} {_hex(b[1][1])} {_hex(b[0][2])} 0 1111 1
{_hex(a[0][3])} {_hex(a[1][2])} {_hex(a[2][1])} {_hex(a[3][0])} {_hex(b[3][0])} {_hex(b[2][1])} {_hex(b[1][2])} {_hex(b[0][3])} 0111 1; initiate display
0 {_hex(a[1][3])} {_hex(a[2][2])} {_hex(a[3][1])} 0 {_hex(b[3][1])} {_hex(b[2][2])} {_hex(b[1][3])} 1011 1; display begins
0 0 {_hex(a[2][3])} {_hex(a[3][2])} 0 0 {_hex(b[3][2])} {_hex(b[2][3])} 1101 1
0 0 0 {_hex(a[3][3])} 0 0 0 {_hex(b[3][3])} 1110 1
0 0 0 0 0 0 0 0 1111 1; last display tic
"""
```

Admittedly, this is not the most elegant solution I could have come up with,
but it was a very quick way to a working program. This reduces the burden of
writing test cases to writing them as python lists.

I am glad we did this, because writing multiple test cases exposed a problem
with our design: when we sent signals on our $$\textbf{B}$$ input lines, these
signals "skipped" values when they reappeared on the registered output lines.
Perhaps this is an obvious obstacle to diagnose for more experienced VLSI
teams, but this was an unexpected head-scratcher for us.

## Resolving $$\mathbf{B}$$ Signal Timing Violations

With some experience splitting work between us, we implemented three strategies
to attempt resolving this issue

1. Increasing our clock period, to see if the design passed at higher values
2. Making a design variant that exposed more internal signals, to see if hold
   time violations were occurring on the $$\textbf{B}$$ signal between
   processing elements.
3. Adjusting longer interconnect nets (e.g., `clk`) to equalize delay between
   distant processing elements

A change to the clock period in our vector files revealed that the design
worked at 100 MHz (much slower than our original 500 MHz target). So, while
one engineer worked on trying to cut this corner closer (a slow process; each
simulation takes 30 minutes), two worked on the other two strategies.

Our testbenches for all designs discussed thus far consist of connecting
schematic I/O pins to the associated component I/O pins. If we wanted to be
thorough, we would include inverters on these lines to
["load" them with capacitance](https://www.chinachipsun.com/load-capacitance/),
but we omitted doing this in later designs to speed up our development cycles.
Adjusting these testbenches to expose internal pins was not a trivial task (the
only way we found to do this was to re-implement the entire component with
these extra pins), and our simulations were already running into disk quota
limits without these extra signals to capture. Hence, this strategy did not
bear any fruit.

It was when we turned to our clock nets when we found a solution that worked
for us: our initial design of the matrix multiplier routed the clock net on a
wire between the second and third row of processing elements, placing the pin
on the left edge. Moving this pin to the center of this wire resolved our
timing violations for all test cases we simulated at the original frequency.

What this means for our design: if larger designs use the matrix multiplier,
they must route their clock wire such that it connects with this clock net at
the indicated position in the middle of the design, instead of following our
prior strategy of "reusing nets wherever possible". Now our design roughly
approximates a [clock h-tree](https://vlsiweb.com/clock-tree-structures/) and
is thus able to control for clock skew by providing similar interconnect
lengths between extant paths.

Given more time, careful investigations into how much timing margin our design
affords could be warranted. However, time was a resource on which we were
short. Instead, we took our design to the course project demo day, knowing it
would pass the cases we were assigned to test.

## Our results

**Power**: Our design consumes ~1.078 mW to produce a matrix multiplication

**Area**: The longest dimension of our design was the width at 118.51 microns

**Delay**: Our clock period is 2 ns

This produces a PAD metric of 30278.5 $$\text{mW} \: \text{um}^2 \: \text{ns}$$.

Informal comparisons with other teams in the course (namely, our friends who
were mutually curious) revealed this to be a fair result. Notably, our design
was larger than some of our competitors but ran at a more aggressive frequency,
producing a comparable PAD. We were, of course, severely outperformed
by teams employing more experienced VLSI layout engineers reaching for their
own graduate degrees, but this was an extremely informative, engaging, and
(dare I say) fun experience that I seek to build on in future courses,
positions, and blog posts.

# Takeaways

Having gone through the project, these are our lessons learned:

1. **Forecasting task timeframes**: Division of work depends on the work
   partitions being relatively equal. Since we did not consult anyone on this
   subject before dividing our work, our divisions ended up unequal at first.
   Taking time to plan this out (as well as find second opinions from subject
   matter experts -- i.e., our T.A.s) would have ameliorated this issue.
2. **Automation and peer development sessions**: Different members of our
   team had learnt different tricks and developed different tools to do
   things. Sharing knowledge within teams is key to speeding up total
   development, particularly where everyone has to do a little bit of
   everything.
3. **Source control**: As we had all done our labs/prior work separately, if we
   had to do this project I would allocate more time to project setup. We had
   difficulty sharing our work with each other that could have been reduced
   with version control (such as `git`, perhaps) and decisions on project
   setup. (We all contributed our work on different projects, so the hierarchy
   we produced required several libraries of standard cells.) One
   "project" library and as few "dependency" libraries as possible would have
   served us best.
4. **Work reviews**: When we complete major milestones it pays off to do quick
   reviews to uncover errors and issues before downstream tasks obfuscate them.
   Our processing element is a key example of this: the initial layout routed
   the $$\textbf{B}$$ pins in the wrong direction, which would have only been
   an obvious issue when designing vector/matrix multipliers. Resolving it
   before moving to these designs meant minimal rework. A better strategy would
   have been to decide/verify layout pin positions together before laying out
   the incorrect placements.
5. **Early and repeated functional verification**: Simulations on schematic
   netlists don't give the parasitics and accuracy that doing it post-layout
   would produce, but it can still expose issues that would require layout
   rework if the simulations weren't run first. The workflow should look like
   this: draw the schematic, then verify. Then, draw the layout and verify.
   Teams can multitask by 'speculatively' drawing the layout while simulations
   finish (something we did with our matrix multiplier) if they acknowledge
   the probability/risk of discarded work (and they have nothing else to do).
6. **Foresight**: The efforts we took to make our future work easier
   single-handedly made the difference between us making and missing our
   deadline. Provided these tangents are short, teams should look to implement
   workflow improvements early to gain the most benefit.
7. **Using DRC and LVS checkers**: This was a story we all had in common. At
   first, we would run DRC on designs to correct wire/component placement
   errors, only to learnt that our layout was wrong in LVS. Then, correcting
   the LVS errors would break DRC. Our project taught us that LVS should be
   prioritized first. (I also used DRC to check incremental design decisions,
   such as "can I place this component here, or is it too close?")

To reiterate, none of us carry much experience in this field. However, these
tips may help other novices and engineers needing refreshers. These are all
lessons I look forward to using in future work.
