\documentclass{wg21}

\usepackage{adjustbox}
\usepackage{enumitem}
\usepackage{epigraph}
\usepackage{fullpage}
\usepackage{listings}
\usepackage{minted}
\usepackage{multirow}
\usepackage{parskip}
\usepackage{soul}
\usepackage{ulem}
\usepackage{xcolor}
\usepackage{tcolorbox}
\usepackage{cprotect}

% For wording updates
\usepackage{changepage}

\frenchspacing

\tcbuselibrary{theorems}
        
\patchcmd{\thebibliography}{\section*{\refname}}{}{}{}

\setlength{\epigraphwidth}{0.9\textwidth}
\setlength{\epigraphrule}{0pt}

\definecolor{proposalBG}{RGB}{147,112,219}

\newcounter{proposalnum}
\newtcbtheorem[use counter=proposalnum]{proposal}
              {Proposal}
              {colback=proposalBG!20!white,colbacktitle={proposalBG!40!white},coltitle={black}}
              {prop}

\newtcbtheorem[auto counter,number within=proposalnum]{subproposal}
              {Proposal}
              {colback=proposalBG!20!white,colbacktitle={proposalBG!40!white},coltitle={black}}
              {subprop}

\title{Interaction between relocation and swap}
\docnumber{D3278R0}
\docdate{\today}  % replace with YYYY-MM-DD on publication
%\docdate{yyyy-mm-dd}
\audience{}
\author{Nina Ranns}{dinka.ranns@gmail.com}

\begin{document}
\maketitle

\begin{abstract}
\noindent
In this paper we analyze the relationship between relocation and swap.
Additionally, we address the points raised in P3236 (%todo: insert link).\end{abstract}

\tableofcontents

\clearpage
\section*{Revision History}

Revision 0

\section{Introduction}

P2786 (https://isocpp.org/files/papers/D2786R6.pdf) and P1144(insert link) both attempt to address relocation and trivial relocation for the purposes of optimiziation. Both papers also define the meaning of relocation.

From P1144
"We define a new verb, "relocate," equivalent to a move and a destroy. For many C++ types, this is implementable "trivially," as a single memcpy."

From P2786 
"— To relocate a type from memory address src to memory address dst means to perform an operation or series of operations such that an object equivalent (often identical) to that which existed at address src
exists at address dst, that the lifetime of the object at address dst has begun, and that the lifetime of the
object at address src has ended."

Many operations within the standard library are morally relocations, such as erase, insert (to do: add other examples), yet they are often implemented in terms of assignment. 
Swap, on the other hand, exchanges values of two existing objects, but can for certain subset of types be implemented as a series of relocations. It is important to note that swap, inherently, isn't a relocation. We do want to enable swap optimisations when we know that the relocation will do "the right thing".

To untangle the problem of relocation and swap, we will analyse both separately, 

\section{Relocation and Trivial relocation}

We argue that all operations that morally perform relocation should be able to benefit from trivial relocation optimizations, according to the definition of trivially relocatable types in P2786 (Unlike P1144, P2786 does not take assignment operation into consideration when determining which type is implicitly trivially relocatable).

It is quite common to view operations like vector::erase and vector::insert as performing relocations. However, the standard specification is opaque about whether these operations are relocating whole objects or just managing values. 

In practice, most standard Lsbrary implementations use assignment when moving values to locations where there are currently objects.   In some cases, this is even implied with both requirements for assignability and statements like
[https://eel.is/c++draft/sequences#deque.modifiers-6]
"Complexity: The number of calls to the destructor of T is the same as the number of elements erased, but the number of calls to the assignment operator of T is no more than the lesser of the number of elements before the erased elements and the number of elements after the erased elements"


For a lot of types, the distinction between assignment to an existing object and construction of a new object into a space occupied by an existing object is non existent, and the implementation which uses and assignment to achieve the relocation of objects can't be distinguished from one that does construction. 
Overall, `vector` is written as if all types behaved consistently when replacement is swapped with assignment, and that is clearly not the case for a number of types that have otherwise reasonable semantics.

We observe two cases of types where assignment and construction (possibly) yield a different result.

1) types for which relocation operations implemented in terms of assignment yields the wrong result

For axample, pmr::string propagates the allocator on move construction, but does not propagate the allocator on move assignment. If relocation operations uses assignment, the resulting allocator may not be the correct one. 

2) types for which it isn't clear what the desired semantic is

Tuple<int&> binds the reference on construction, but assigns through the reference in assignment. It is generally unclear what should happen for the objects of tuple<int&> relocated during something like vector::erase.

We argue that neither category of types outlined above prevents library implementors to take advantage of trivial relocation within the operations that are morally relocations : it will give the correct behaviour for former types, while latter types already have an unclear semantic and it's questionable whether the new semantic would be wrong. We do not propose adding additional specifications to the library, we want to leave the choice on how and when these operations are optimised to the library authors.

We also argue that we do not want to pesimize future implemenations which are specified in terms of relocation because we have not distinguishied between constriction and assignment semantics so far. 

Note that the implementors do have a choice to preserve the behaviour of existing methods for the types outlined above in the same way they might do for swap. 

\section{swap}

When talking about swap, the standard talks about swapping *values*. For certain subset of types, a value is also the object representation. If value and object representation are the same, a swap can also be implemented as a set of relocation operations. Note that this is almost the opposite to the former case 
- in vector::insert we can in some cases implement a relocation using assignment
- in swap, we can in some cases implement assignment using relocation


Two things need to hold for swap to be able to be reduced to relocation
1) type needs to be relocatable
2) the semantic should not change if we use relocation instead of assignment

P1144 recognises the benefits of optimising swap to use relocation where possible, and P3236 provides the library author's request to reason about when swap can be implemented in terms of relocation. 

We argue that the way forward is to carve out types for which construction+destruction is equivalent to assignment. We refer to these types within this paper as replacable types. A trait is_replacable<T> could be provided. One possible specification of such a trait is one which is true for types where no copy constructor, asignment operator, or destructor is virtual or user provided, and false otherwise. A type could also specialise the trait to explicitly say whether the equivalence holds. Note that this paper is not proposing the exact semantics of such trait, we are simply demonstrating what such trait might look like. 


Implemeting swap would then be a case of writing a specialisation for types for which both is_trivially_relocatable and is_replacable holds. Note that is_replacable could also be used in other situations where one needs to understand whether assignment has different semantics to copy construction.

This solves the problem of tuple<int> vs tuple<int&> problem raised by P3236.

Additional problem with implementing swap in terms of relocation is the fact that trivial relocation is usually implemented in a way that ends and starts lifetimes of object that have been swapped. This has consequences in possibly adding UB where UB doesn't exist when swap is implemented im terms of assignment. In order to solve that problem, we would need a magic function that swaps the objects using relocation, but does not end lifetime of either of the object. 

Both a version of the above trait and the magic function are proposed in (%todo:insert number)


\section{virality of relocate}

When allowing user to mark their type as relocatable, there are several options

1) check whether all the subobjets of such type are relocatable, and prevent marking the top level type as relocatable if some parts are not relocatable
2) allow the author to mark the top level as relocatable only if all of the subobjects are not explicitly marked as not relocatable
3) allow the author to mark the top level as relocatable in all cases

We recognise that 2 and 3 allow for an easier transition. However, option 1 is the safest option. We can consider relaxing the rules at a later point to allow for more cases when a type can be marked as relocatable, but we can not make the rules more strict.

Note that the question of when to allow a type to be marked as relocatable based on the relocatibility of its subobjects is a separate question to whether relocation takes assignment into account. 

\section{attribute vs core feature}

Backwards compatibility can be achieved by 

#ifdef __cpp_relocation_macro
relocatable
#endif

%TODO add other reasons why core feature is better than an attribute.


Note that the question of whether we should have an attribute or a core language feature to mark a type as relocatable is a separate question to whether we should take assignment into account for relocation, and a separate question to whether we should allow marking a type relocatable if all its subobjects are not relocatable.

\section{conclusion}

Separating relocation from assignment and having a trait that specifies whether assignment can be replaced with relocation offers the most benefit

- new relocation operations are not limited by the assignment operator of the type
- library implementors have a choice on choosing which way to optimise for already existing relocation operations, allowing them not to pesimise based on assignment operator
- swap can be correctly implemented 

It is tempting to solve both relocation operations and swap in one go. However, we propose that it's better not to pesimize all curent and future relocation operations because of swap, and to continue the efforts in allowing swap to be optimized too by pursuing the direction of defining what a replacable type is and making sure we do not introduce additional UB.

We should also come to an explicit agreement on attribute vs core feature aspect of relocation, and whether we should allow a type to be marked relocatable if its subobjects are not relocatable. While aspects of P2786 were discussed in EWG session in Tokyo 2024, no explicit polls have been taken. 

\appendix

\section*{Bibliography}
\bibliographystyle{wg21} 
\bibliography{wg21-bib, D3272R0-bib}

\end{document}


