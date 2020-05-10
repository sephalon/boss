**************************************
Board State Distribution System (BoSS)
**************************************

aka *Board State Detection and Reporting*

This project aims at providing a non-invasive DUT board state detection and 
reporting, e.g. for monitoring and test protocol generation purposes.
The components of the system is distributed across multiple networked nodes
to balance compute tasks, e.g. state detection (computer vision), and 
information visualition as well as logging.


.. _summary:

Summary
-------

The idea for this project was born when more and more lab setups have been
migrated to a labgrid environment to enable remote control and interconnection.
This, e.g. enables remote console connectivity via UART / SSH or power / reset 
control.
However, the missing piece is a way to check the optical indicators of a board,
such as LEDs.


.. _this_doc:

This document
-------------

This document is a working document summarizing the current state of thaughts 
mostly on conceptual and architectural levels without any details about the 
implementation.

However, some ideas about details are available and mentioned in 
subsection implementation_ideas_.

The document is written to be supported by [sphinx]_ to render a pdf, HTML or
epub version.

This documents source is available on github_.

.. _github: https://github.com/missinglinkelectronics/boss


.. _boss_overview:

Overview about BoSS
-------------------

When thinking about LED state detection a couple of more states of additiona 
board components may be extracted and reported:

#. Visuals

    #. LED states

        #. state: on, off, blinking
        #. color: green, red, blue, yellow, (RGB value)
        #. blinking: rate if blinking and rate detectable

    #. Multiple Segment Displays (e.g. multiple LEDs)

        #. Char / Number Displays
           
            #. positions: number of characters
            #. segments: number of segments per character (incl. dots, etc.)
            #. string: Character(s) displayed
            #. segment state: which segments are turned on

        #. Bar Displays

            #. positions: number of bars
            #. segments: number of segments per bar
            #. position: max position currently on
            #. segment state: list of all segments turned on
    
#. Switches

    #. DIP switch states

        #. positions: number of positions
        #. state: list of single switch states (on, off)

    #. Rotary switches

        #. positions: number of positions
        #. state: current position (number)

    #. Flip switches

        #. positions: number of positions
        #. state: current position
    
#. Jumper states

    #. state: on, off, position

The boards itself are described by an xml file, which actually is an adapted
*inkscape*. This file comprises a picture of the board and a node for each 
board component to be monitored. For each board revision / version the 
information repository serves a seperate board description, e.g. an xml file. 
Each components to be tracked, is described within the board description and 
comprises information about the components, i.e.

#. location,
#. dimension and
#. type. 

Additionally a model for board position and orientation matching could also be 
supplied as part of the board description.
To enable humans identifing the board additional components that also might be
tracked by the software tool for matching the board type, revision and serial
number are specified with location and values to identify the board(s). A 
serial number is optional and might only be reported by the tool, if possible.
The type and revision usually makes sense to have.
Additional information like a comment or links to the manufacturer or vendor 
might be added as well within a list of arbitrary attributes, which are not 
used by the tools.


.. _entities_overview:

Entities
--------

The overall system comprises of several entities profiding and consuming 
subservices.

    #. Board Description Generator (BDG)
    
    #. Board Information Provider (BIP)
    
    #. Board State Provider (BSP)
    
    #. Board State Coordinator (BSC)
    
    #. Board State Visualizer (BSV)
    
    #. Board State Logger (BSL)

These entities work together providing all the information for contineous 
board state tracking and reporting. To enable the communication among the 
entities appropriate protocols are needed, such as

#. Board Information Distribtuion Protocol (BIDP)
#. Board State Distribtuion Protocol (BSDP)
#. Board State Component Discovery Protocol (BSCDP)

The following sections provide an overview about the entities and its purposes 
as well as the protocols and some more general sections, e.g. about 
implemnentation ideas and license ideas.


.. _data_flow:

Board State Data Flow
---------------------

The overall data flow of the BOSS is shown in figure img_boss_data_flow_. 
The data flow of the quasi static data, shown in green provides general data
about the board. It is communicated from the *BIP* to the board state 
providers and consumers via the *BIDP*. This data is created with the help of 
*BDG* and distributed to the other entities via the *BIP*. The *BSC* offers 
the information about how to connect to the board state provider, the *BSP*, 
to the *BSV*, *BSL* or additional state information consumers. The most 
dynamic data, which is the board state change events itself is served by *BSP* 
to the board state consumers *BSV and *BSL*. This information is distributed 
via the *BSDP*.

.. _img_boss_data_flow:

.. image:: doc/fig/boss_data_flow.png


.. _entity_bdg:

Board Description Generator (BDG)
+++++++++++++++++++++++++++++++++

The *Board Description Generator*, shor *BDG* is a tool that enables to 
generate a board descriptions, which will be a file comprising all necessary 
data to identify a board both by humans and PCs, e.g. via type and revision 
tags for humans as well as a model for automated visual matching by a software.

The *BDG* provides a way to enter the respective information and package it up
for distribtuion via a board information provider, see entity_bip_.

The *BDG* could be implemented as an *inkscape* [inkscape]_ plugin, which 
provides tools to verify that all necessary attributes are specified and helps 
a user to add these fields, e.g. the tracked component attributes.

the compiled iformation will be provided from the *BDG* to the *BIP* via the 
*BIDP* or another suitable protocol not yet described within the document. It 
might also rely on manual steps as this step is not executed as often as the 
others and is also not part of the automated process.


.. _entity_bip:

Board Information Provider (BIP)
++++++++++++++++++++++++++++++++

The BIP implements a service providing general information about boards.
It might be a generally available service, such as a public repository and
/ or a local repository.

Existing protocols, such as git could be used to easily implement necessary
versioning and distribution services. Additionally the decentralised approach
is valuable. Also publicly available services, such as github could be used for
collaboratively serve board information for widely used boards.

Within the logger / visualizer entities it should be possible to use multiple 
repositories to, at the same time, use global and local repositories at the 
same time to leverage publicely available information and solely internally 
available information as well.


.. _entity_bsp:

Board State Provider (BSP)
++++++++++++++++++++++++++

The *BSP* dynamically extracts board state information from the board, e.g. 
via computer vision algorithms that analyse camera images taken from the board 
of interest. This way LED, switch and jumber states can be derived.

Based on image processing a couple of important algorithms need to be 
implemented within the image processing pipeline:

#. camera image adjustment w.r.t. general image parameters, i.e. white
   balance, contrast, etc

#. board position and orientation matching, e.g. based on a supplied model
    (board features) from the BSP implementing AI, HOG + SVM [hog_svm], SIFT
    or other more classical openCV algorithms

#. generation of a board state representation

#. generation of board state state events

#. serializtion into the *BSDP* distribution of board state events to the
   board state consumers, e.g. *BSB* and *BSL*

the actual state detection is best modularized depending on the components to 
to be tracked, e.g. a module for tracking LEDs, another one for bar displays 
(possible build on top of the LED state tracking module), another one for 
jumpers and so on.

The entity of the *BSP* is the core compute element of the *BoSS* as it 
provides the actual board state information.


.. _entity_coordinator:

Board State Coordinator (BSC)
+++++++++++++++++++++++++++++

The *BSC* is the entity that provides the way of service discovery as it 
implements a dictionary of service providers to be accessed by the service 
consumers.
This way the consumers know how to connect to the service provider that offers 
the information needed.

As already noted, this service is initially thaught to be implemented via 
labgrid. As the idea of this system is to complement the existing services 
offered by labgrid and following the labgrid approach of grouping and managing 
services, rather than re-implementing them again.


.. _entity_bsv:

Board State Visualizer (BSV)
++++++++++++++++++++++++++++

The *BSV* is the tool the user will mostly use to interact with the *BoSS* as 
it actually presents the gathered and distributed information either via a 

#. command line interface (CLI) or 
#. a graphical user interface (GUI).

The former CLI interface may list state changes or provide a cleaner way to 
present information, e.g. via a ncurses based CLI, see [ncurses].
The latter GUI interface may use the board picture provided by the BIDPEgk4

Optionally a board state consumer may request the provider to send the RAW 
video stream for manual visual inspection.


.. _entity_bsl:

Board State Logger (BSL)
++++++++++++++++++++++++

The *BSL* is a board state consumer dedicated to providing logging facilities, 
especially for regression test setups, where automated test execution systems 
drive the board and the results are automatically analysed and archived for 
later inspection.


.. _protocol_bidp:

Board Information Distribtuion Protocol (BIDP)
++++++++++++++++++++++++++++++++++++++++++++++

The *BIDP* distributes the general board information used by the board state 
provider and consumers. It may reuse existing and well known protocols, such 
as git, ftp, webdav or even local folders / network attached drives.


.. _protocol_bsdp:

Board State Distribtuion Protocol (BSDP)
++++++++++++++++++++++++++++++++++++++++

The *BSDP* carries the actual board state served ba the board state providers 
and consumed by the consumers. This protocol is the core protocol and should 
be formally specified to provide automatic validation of sent and received 
messages. So, the messages may employ JSON, see [JSON] or XML [XML] based 
data serialization methodologies to encode the data into the messages. The 
*BSDP* may provide state updates, i.e. events, as soon as a new state has 
been detected. This means at any time, a stream of events define the whole 
current full board state. After an intialization for connection setup an 
initial board state, covering all components should be transferred, followed 
by a stream of events, that are pushed to the board state consumer. However, 
in case a consumer looses track of the current event stream, the full state 
report could be requested from the *BDP*.


.. _protocol_bscdp:

Board State Distribution Component Discovery Protocol (BSCDP)
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

The *BSCDP* initially is thaught to be implemented by using labgrid, [labgrid] 
infrastrucure, i.e. its services and its way to describe places for 
information description and distribution.


.. _implementation_ideas:

Implementation Ideas
--------------------

The protocols used should be formally specified, e.g. via openAPI to enable
autometed checks as well as generated stubs for interacting entities.

The idea is that the entities are implemented in *Python 3* to leverage
existing technology, especially for the image processing part.

Generally, existing entities, components and protocols should be used to 
minimize the amount of effort needed to create and maintain the project.


.. _implementation_ideas_boards:

First Supported Boards
++++++++++++++++++++++

The first boards to be supported should be widely used boards and at least one 
shoult provide at least on example of each component supported by the state 
providers and consumers. Such boards could e.g. be:

#. Raspberry PI: widely used cheap and simple board 
   (any version, at best one of the most sold)
#. Xilinx ZCU102: A widely used example of a more complex board
   (any version, at best one of the most sold)


.. _license_ideas:

License Ideas
-------------

The ideas is to use a license that does not enforce a strong copy left, but 
encourages the continuation of the project as an open source project.
This is why the LGPL v2.1 has been chosen for the repo.

However, if an initial discussion shows contradicting yet convincing ideas in 
favor of a more open or similar licensing scheme, the path for a licensing 
change is wide open.

.. this is intended to be rendered as a reference section 

.. [inkscape] Inkscape 1.0, https://inkscape.org/
.. [sphinx]   Sphinx Documentation, https://www.sphinx-doc.org/
.. [hog_svm]  Vehicle detection with HOG and Linear SVM, https://medium.com/@mithi/vehicles-tracking-with-hog-and-linear-svm-c9f27eaf521a
.. [sift]    Scale-invariant fetature transform, https://de.wikipedia.org/wiki/Scale-invariant_feature_transform
.. [labgrid]   labgrid Documentation, https://labgrid.readthedocs.io/en/latest/
.. [ncurses] NCURSES, https://invisible-island.net/ncurses/announce.html#h2-overview
