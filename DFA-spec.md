# **DFA: The Dagstuhl Format for Assembly**

*J. Chin, R.Durbin, G. Myers*  
*Aug. 31, 2016*

## PROLOG

DFA is a generalization of GFA that allows one to specify an assembly graph in either less detail,
e.g. just the topology of the graph, or more detail, e.g. the multi-alignment of reads giving
rise to each sequence.  We further designed it to be a suitable representation for a string
graph at any stage of assembly, from the graph of all overlaps, to a final resolved assembly
of contig paths with multi-alignments.  Apart from meeting our needs, the extensions also
supports other assembly and variation graph types.

We further want to emphasize that we are proposing a core standard.  As will be seen later in
the technical specification, the format is **extensible** in that additional description lines
can be added and additional SAM tags can be appended to core description lines.

In overview, an assembly is a graph of vertices called **segments** representing sequences
that are connected by **edges** that denote local alignments between the vertex sequences.
At a minimum one must specify the length of the sequence represented, further specifying the
actual sequence is optional.  In the direction of more detail, one can optionally specify a
collection of external sequences, called **fragments**, from which the sequence was derived (if
applicable) and how they multi-align to produce the sequence.  Similarly, the specification
of an edge need only describe the range of base pairs aligned in each string, and optionally
contain a [**Dazzler-trace**](http://wp.me/p4o3kW-88)
or a [**CIGAR string**](https://samtools.github.io/hts-specs/SAMv1.pdf)
to describe the alignment of the edge.  Traces are a
space-efficient Dazzler assembler concept that allow one to efficiently reconstruct an
alignment in linear time, and CIGAR strings are a SAM concept explicitly detailing the
columns of an alignment.  Many new technologies such a Hi-C and BioNano maps organize segments
into scaffolds along with traditional data sets involving paired reads, and so a **gap** edge
concept is also introduced so that order and orientaiton between disjoint contigs of an
assembly can be described.  Finally, one can describe and attach a name to any **path** or
**subgraph** in the encoded string graph.

## GRAMMAR

```
<spec>     <- ( <header> <segment> | <fragment> | <edge> | <gap> | <group> )+

<header>   <- H {VN:Z:2.0} {TS:i:<trace spacing>}

<segment>  <- S <sid:id> <slen:int> <sequence>

<fragment> <- F <sid:id> [+-] <external:id>
                  <sbeg:pos> <send:pos> <fbeg:pos> <fend:pos> <alignment>

<edge>     <- E <eid:id> <sid1:id> [+-] <sid2:id>
                         <beg1:pos> <end1:pos> <beg2:pos> <end2:pos> <alignment>

<gap>      <- G <eid:id> <sid1:id> [+-] <sid2:id>
                         <disp:pos> <var:int>

<group>    <- [UO] <pid:id> <item>([ ]<item>)*

    <id>        <- [!-~]+
    <item>      <- <sid:id> | <eid:id> | <pid:id>
    <pos>       <- {$}<int>
    <sequence>  <- * | [!-~]+
    <alignment> <- * | <trace array> | <CIGAR string>

      <CIGAR string> <- ([0-9]+[MX=DIP])+
      <trace array>  <- {<int>:}<int>(,<int>)*
```

In the grammar above all symbols are literals other than tokens between <>, the derivation
operator <-, and the following marks:

  * {} enclose an optional item
  * | denotes an alternative
  * * zero-or-more
  * + one-or-more
  * [] a set of one character alternatives.

Like GFA, DFA is tab-delimited in that every lexical token is separated from the next
by a single tab.

Each descriptor line must begin with a letter and lies on a single line with no white space
before the first symbol.   The tokens that generate descriptor lines are \<header\>, \<segment\>,
\<fragment\>, \<edge\>, \<gap\>, and \<group\>.
Any line that does not begin with a recognized code (i.e. H, S, F, E, G, or P) can be ignored.
This will allow users to have additional descriptor lines specific to their special processes.
Moreover, the suffix of any DFA descriptor line may contain any number of user-specific SAM
tags which are ignored by software designed to support the core standard.

## SEMANTICS

The **header** contains an optional 'VN' SAM-tag version number, 1.0, and an optional
'TS' SAM-tag specifying the default trace point spacing for any Dazzler traces specified
to accelerate alignment computation.
Any number of header lines containing SAM-tags may occur.

A **segment** is specified by an S-line giving a user-specified ID for the
sequence, its length in bases, and the string denoted by the segment or * if absent.
The sequence is typically expected to be bases or IUPAC characters, but GFA2 places
no restriction other than that they be printable characters other than space.
The length does not need to be the actual length of the sequence, if given, but is rather
an indication to a drawing program of how long to draw the representation of the segment.
The segment sequences and any CIGAR strings referring to them if present follow the
*unpadded* SAM convention.

**Fragments**, if present, are encoded in F-lines that give (a) the segment they belong to, (b) the
orientation of the fragment to the segment, (c) an external ID that references a sequence
in an external collection (e.g. a database of reads or segments in another DFA or SAM file),
(d) the interval of the vertex segment that the external string contributes to, and (e)
the interval of the fragment that contributes to to segment.  One concludes with either a
trace or CIGAR string detailing the alignment, or a \* if absent.

**Edges** are encoded in E-lines that in general represent a local alignment between arbitrary
intervals of the sequences of the two vertices in question. One gives first an edge ID and
then the segment ID’s of the two vertices and a + or – sign between them to indicate whether
the second segment should be complemented or not.  An edge ID does not need to be unique,
*unless* you plan to refer to it in a P-line (see below).  This
allows you to use something short like a single *-sign as the id when it is not needed.

One then gives the intervals of each segment that align, each as a pair of *positions*.  A position
is either an integer or the special symbol $ followed immediately by an integer.
If an integer, the position is the distance from the *left* end of the segment,
and if $ followed by an integer, the distance from the *right* end
of the sequence.  This ability to define a 0-based position from either end of a segment
allows one to conveniently address end-relative positions without knowing the length of
the segments.  Note carefully, that the segment and fragment intervals in an F-line are
also positions.

Position intervals are always intervals in the segment in its normal
orientation.  If a minus sign is specified, then the interval of the second segment is
reverse complemented in order to align with the interval of the first segment.  That is,
<code>S s1 - s2 b1 e1 b2 e2</code> aligns s1[b1,e1] to the reverse complement of s2[b2,e2].

A field for a 
[**CIGAR string**](https://samtools.github.io/hts-specs/SAMv1.pdf)
or [**Dazzler-trace**](http://wp.me/p4o3kW-88)
describing the alignment is last, but may be absent
by giving a \*.  One gives a CIGAR string to describe an exact alignment relationship between
the two segments.  A trace string by contrast is given when one simply wants an accelerated
method for computing an alignment between the two intervals.
A trace is a list of integers separated by commas, each integer giving the # of characters in
the second segment to align to the next *TS* characters in the first segment where
the *TS* is either the default trace spacing given in a header with the TS SAM-tag, or it
is an optional prefixing <code><int>:</code> in front of the trace list. 
If a \* is given as the alignment
note that it is still possible to compute the implied alignment by brute force.

The DFA concept of edge generalizes the link and containment lines of GFA.  For example a GFA
edge which encodes what is called a dovetail overlap (because two ends overlap) is simply a DFA
edge where end1 = $0 and beg2 = 0 or beg1 = 0 and end2 = $0.   A GFA containment is
modeled by the case where beg2 = 0 and end2 = $0 or beg1 = 0 and end1 = $0.  The figure
below illustrates:

![Illustration of position and edge definitions](DFA.Fig1.png)

Special codes could be adopted for dovetail and containment relationships but the thought is
there is no particular reason to do so, the use of the special characters for terminal positions
makes their identification simple both algorithmically and visually, and the more general
scenario allows interesting possibilities.  For example, one might have two haplotype bubbles
shown in the “Before” picture below, and then in a next phase choose a path through the
bubbles as the primary “contig”, and then capture the two buble alternatives as a vertex
linked with generalized edges shown in the “After” picture.  Note carefully that you need a
generalized edge to capture the attachment of the two haplotype bubbles in the “After” picture.

![Example of utility of general edges](DFA.Fig2.png)
 
While one has graphs in which vertex sequences actually overlap as above, one also frequently
encounters models in which there is no overlap (basically edge-labelled models captured in a
vertex-labelled form).  This is captured by edges for which beg1 = end1 and beg2 = end2 (i.e.
0-length overlap)!

While not a concept for pure DeBrujin or long-read assemblers, it is the case that paired end
data and external maps often order and orient contigs/vertices into scaffolds with
intervening gaps.  To this end we introduce a **gap** edge described in G-lines that give the
estimated gap distance between the two segment sequences and the variance of that estimate
or 0 if no estimate is available.  The first segment is in always in the normal orientation.
If the displacement is an integer g (no $-prefix), then g is the estimated distance from the
*end of the first segment forward* to the second fragment in the orientation specified by the sign.
If the displacement is $g, then the g is the estimated distance from the *start of the
first segment backward* to the second fragment in the orientation specified by the sign.
Relationships in E-lines are fixed and known, where as
in a G-line, the distance is an estimate and the line type is intended to allow one to
define assembly **scaffolds**.

A **group** encoding on a U- or O-line allows one to name and specify a subgraph of the
overall graph.
Such a collection could for example be hilighted by a drawing program on
command, or might specify decisions about tours through the graph.  U-lines encode
*unorder* collections and O-lines encode *ordered* collection (defined in the next paragraph).
The remainder of
the line then consists of a name for the collection followed by a non-empty list of ID's
referring to segments and/or edges that are *separated by single spaces* (i.e. the list is
in a single column of the tab-delimited format).  U/O-lines with the same name are considered
to be concatenated together in the order in which they appear, and a group list may refer
to another group recursively.

An unordered collection defined in a U-line refers to
the subgraph induced by the vertices and edges in the collection (i.e. one adds all edges
between a pair of segments in the list and one adds all segments adjacent to edges in the
list.)   An ordered collection defined in an O-line captures paths in the graph consisting of
the listed objects
and the implied adjacent objects between consecutive objects in the list (e.g.
the edge between two consecutive segments, the segment between two consecutive edges, etc.)

There is a single name-space for the set of all id's, whether for segments, edges, or groups.
All segments and group id's must be unique, but any number of edges can share the same ID
(e.g. something simple like *), as long as they do not need to be referred to in a group
list.  Because there can be more than one edge between a given pair of segments, a pair
of segments does not always suffice to uniquely identify an edge for a path, and so one
must in thes cases refer to the desired edge whose id must be unique.  Every id in a
group list must refer to a unique ID, it is an error otherwise.

## BACKWARD COMPATIBILITY WITH GFA

DFA is a super-set of GFA, that is, everything that can be encoded in GFA can be encoded
in DFA, with a relatively straightforward transformation of each input line.

The syntactic conventions of DFA are identical to GFA, so upgrading a GFA parser to
a DFA parser is relatively straight forward.  Each description line begins
with a single letter and has a fixed set of fields that are tab-delimited.  The changes
are as follows:

1. There is an integer length field in S-lines.

2. The L- and C-lines have been replaced by a consolidated E-line.

3. The P-line has been replaced with U- and O-lines that encode subgraphs and paths, respectively,
   and can take edge id's, obviating the need for orientation signs and alignments between segments.

4. There is a new F-line for describing multi-alignments and a new G-line for describing scaffolds.

5. Alignments can be trace length sequences as well as CIGAR strings.

6. Positions have been extended to include a notation for 0-based position with respect
   to *either* end of a segment (the default being with respect to the beginning).
