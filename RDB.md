Relational Databases
====================

We now examine how it is possible to put game data into a database:
First we must give ourselves a level file to work with. We’re not going
to go into the details of the lowest level of how we utilise large data
primitives such as meshes, textures, sounds and such. For now, think of
assets as being the same kind of data as strings[^1].

We’re going to define a level file for a game where you hunt for keys to
unlock doors in order to get to the exit room. The level file will be a
list of different game objects that exist in the game, and the
relationships between the objects. First, we’ll assume that it contains
a list of rooms (some trapped, some not), with doors leading to other
rooms which can be locked. It will also contain a set of pickups, some
that let the player unlock doors, some that affect the player’s stats
(like health potions), and all the rooms have lovely textured meshes, as
do all the pickups. One of the rooms is marked as the exit, and one has
a player start point.

~~~~ {caption="A" setup="" script=""}
// create rooms, pickups, and other things.
m1 = mesh( "filename" )
m2 = mesh( "filename2" )
t1 = texture( "textureFile" )
t2 = texture( "textureFile2" )
k1 = pickup( TYPE_KEY, m5,t5, TintColourGold )
k2 = pickup( TYPE_POTION, m4,t4, null )
r1 = room( worldPos(0,0), m1, t1, hasPickup(k1)+hasTrap(true,10hp)+hasDoorTo(r2) )
r2 = room( worldPos(20,5), m2,t2, hasDoorTo(r3,lockedWith(k1)+isStart(worldPos(22,7) )
r3 = room( worldPos(30,-5), m3,t3, isExit() )
~~~~

[src:roomscript]

In the first step, you take all the data from the object creation
script, and fit it into rows in your table. To start with, we create a
table for each type of object, and a row for each instance of those
obejcts. We need to invent columns as necessary, and use NULL everywhere
that an instance doesn’t have that attribute/column.

[h]Meshes\

  MeshID   MeshName
  -------- -------------
  m1       “filename”
  m2       “filename2”

Textures\

  TextureID   TextureName
  ----------- ----------------
  t1          “texturefile”
  t2          “texturefile2”

Pickups\

  MeshID   TextureID   PickupType   ColourTint   PickupID
  -------- ----------- ------------ ------------ ----------
  m5       t5          KEY          Gold         k1
  m6       t6          POTION       (null)       k2

Rooms\

  RoomID   MeshID   TextureID   WorldPos   Pickup1   Pickup2   ...
  -------- -------- ----------- ---------- --------- --------- -----
  r1       m1       t1          0,0        k1        k2        ...
  r2       m2       t2          20,5       (null)    (null)    ...
  r3       m3       t3          30,-5      (null)    (null)    ...

  ...   Trapped   DoorTo   Locked   Start    Exit
  ----- --------- -------- -------- -------- -------
  ...   10hp      r2       (null)   (null)   false
  ...   (null)    r3       k1       22,7     false
  ...   (null)    (null)   (null)   (null)   true

[tab:initialtables]

Once we have taken the construction script and generated tables, we find
that these tables contain a lot of nulls. The nulls in the rows replace
the optional content of the objects. If an object instance doesn’t have
a certain attributes then we replace those features with nulls. When we
plug this into a database, it will handle it just fine, but storing all
the nulls seems unnecessary and looks like it’s wasting space. Present
day database technology has moved towards keeping nulls to the point
that a lot of large, sparse table databases have many more null entries
than they have data. They operate on these sparsely filled tables with
functional programming techniques. We, however should first see how it
worked in the past with relational databases. Back when SQL was first
invented there were three known stages of data normalisation (there
currently are six recognised numbered normal forms, plus BoyceCodd
(which is a stricter variant of third normal form), and Domain Key
(which, for the most part defines a normalisation which requires using
domain knowledge instead of storing data).

Normalisation
-------------

First normal form requires that every row be distinct and rely on the
key. We haven’t got any keys yet, so first we must find out what these
are.

In any table, there is a set of columns that when compared to any other
row, are unique. Databases guarantee that this is possible by not
allowing complete duplicate rows to exist, there is always some
differentiation. Because of this rule, every table has a key because
every table can use all the columns at once as its primary key. For
example, in the mesh table, the combination of meshID and filename is
guaranteed to be unique. The same can be said for the textureID and
filename in the textures table. In fact, if you use all the columns of
the room table, you will find that it can be used to uniquely define
that room (obviously really, as if it had the same key, it would in fact
be literaly describing the same room, but twice).

Going back to the mesh table, because we can guarantee that each meshID
is unique in our system, we can reduce the key down to just one or the
other or meshID or filename. We’ll choose the meshID as it seems
sensible, but we could have chosen the filename[^2].

If we choose textureID, pickupID, and roomID as the primary keys for the
other tables, we can then look at continuing on to first normal form.

[h]Meshes\

<span>ll</span> **<span>MeshID</span>&MeshName\
m1&“filename”\
m2&“filename2”\
**

Textures\

<span>ll</span> **<span>TextureID</span>&TextureName\
t1&“texturefile”\
t2&“texturefile2”\
**

Pickups\

<span>lllll</span>
**<span>PickupID</span>&MeshID&TextureID&PickupType&ColourTint\
k1&m5&t5&KEY&Gold\
k2&m6&t6&POTION&(null)\
**

Rooms\

<span>lllllll</span>
**<span>RoomID</span>&MeshID&TextureID&WorldPos&Pickup1&Pickup2&...\
r1&m1&t1&0,0&k1&k2&...\
r2&m2&t2&20,5&(null)&(null)&...\
r3&m3&t3&30,-5&(null)&(null)&...\
**

  ...   Trapped   DoorTo   Locked   Start    Exit
  ----- --------- -------- -------- -------- -------
  ...   10hp      r2       (null)   (null)   false
  ...   (null)    r3       k1       22,7     false
  ...   (null)    (null)   (null)   (null)   true

### 1^st^ Normal Form

First normal form can be described as making sure the tables are not
sparse. We require that there be no null pointers and that there be no
arrays of data in each element of data. This can be performed as a
process of moving the repeats and all the optional content to other
tables. Anywhere there is a null, it implies optional content. Our first
fix is going to be the Pickups table, it has an optional ColourTint
element. We invent a new table PickupTint, and use the primary key of
the Pickup as the primary key of the new table.

[h]Pickups\

<span>llll</span> **<span>PickupID</span>&MeshID&TextureID&PickupType\
k1&m5&t5&KEY\
k2&m6&t6&POTION\
**

\
PickupTint\

<span>ll</span> **<span>PickupID</span>&**<span>ColourTint</span>\
k1&Gold\
****

two things become evident at this point, firstly that normalisation
appears to create more tables and less columns in each table, secondly
that there are only rows for things that matter. The latter is very
interesting as when using Object-oriented approaches, we assume that
objects can have attributes, so check that they are not NULL before
continuing, however, if we store data like this, then we know everything
is not NULL. Let’s move onto the Rooms table: make a new table for the
optional Pickups, Doors, Traps, and Start.

[h]Rooms\

<span>lllll</span> **<span>RoomID</span>&MeshID&TextureID&WorldPos&Exit\
r1&m1&t1&0,0&false\
r2&m2&t2&20,5&false\
r3&m3&t3&30,-5&true\
**

\
Room Pickup\

<span>lll</span> **<span>RoomID</span>&Pickup1&Pickup2\
r1&k1&k2\
**

\
Rooms Doors\

<span>lll</span> **<span>RoomID</span>&DoorTo&Locked\
r1&r2&(null)\
r2&r3&k1\
**

\
Rooms Traps\

<span>ll</span> **<span>RoomID</span>&Trapped\
r1&10hp\
**

\
Start Point\

<span>ll</span> **<span>RoomID</span>&Start\
r2&22,7\
**

We’re not done yet, the RoomDoors table has a null in it, so we add a
new table that looks the same as the existing RoomDoorTable, but remove
the row with a null. We can now remove the Locked column from the
RoomDoor table.

[h]Rooms Doors\

<span>ll</span> **<span>RoomID</span>&DoorTo\
r1&r2\
r2&r3\
**

\
Rooms Doors Locks\

<span>lll</span> **<span>RoomID</span>&DoorTo&Locked\
r2&r3&k1\
**

All done. But, first normal form also mentions no repeating groups of
columns, meaning that the PickupTable is invalid as it has a pickup1 and
pickup2 column. To get rid of these, we just need to move the data
around like this:

[h]Room Pickup\

<span>ll</span> **<span>RoomID</span>&**<span>Pickup</span>\
r1&k1\
r1&k2\
****

Notice that now we don’t have a unique primary key. This is an error in
databases as they just have to have something to index with, something
to identify a row against any other row. We are allowed to combine keys,
and for this case we used the all column trivial key. Now, let’s look at
the final set of tables in 1NF:

[h]Rooms\

<span>lllll</span> **<span>RoomID</span>&MeshID&TextureID&WorldPos&Exit\
r1&m1&t1&0,0&false\
r2&m2&t2&20,5&false\
r3&m3&t3&30,-5&true\
**

\
Room Pickup\

<span>ll</span> **<span>RoomID</span>&**<span>Pickup</span>\
r1&k1\
r1&k2\
****

\
Rooms Doors\

<span>ll</span> **<span>RoomID</span>&DoorTo\
r1&r2\
r2&r3\
**

\
Rooms Doors Locks\

<span>lll</span> **<span>RoomID</span>&DoorTo&Locked\
r2&r3&k1\
**

\
Rooms Traps\

<span>ll</span> **<span>RoomID</span>&Trapped\
r1&10hp\
**

\
Start Point\

<span>ll</span> **<span>RoomID</span>&Start\
r2&22,7\
**

The data, in this format takes much less space in larger projects as the
number of NULL entries would have only increased with increased
complexity of the level file. Also, by laying out the data this way, we
can add new features without having to revisit the original objects. For
example, if we wanted to add monsters, normally we would not only have
to add a new object for the monsters, but also add them to the room
objects. In this format, all we need to do is add a new table:

[h]Monster\

<span>llll</span> **<span>MonsterID</span>&Attack&HitPoints&StartRoom\
M1&2&5&r1\
M2&2&5&r3\
**

And now we have information about the monster and what room it starts in
without touching any of the original level data.

### Reflections

What we see here as we normalise our data is a tendency to add
information when it becomes necessary, but at a cost of the keys that
associate the data with the entity. Looking at many third party engines
and APIs, you can see parallels with the results of these
normalisations. It’s unlikely that the people involved in the design and
evolution of these engines took their data and applied database
normalisation techniques, but sometimes the separations between object
and componets of objects can be obvious enough that you don’t need a
formal technique in order to realise some positive structural changes.

In some games the entity object is not just an object that can be
anything, but is instead a specific subset of the types of entity
involved in the game. For example, in one game there might be an object
type for the player character, and one for each major type of enemy
character, and another for vehicles. This object oriented approach puts
a line, invisilbe to the player, but intrusive to the developer, between
classes of object and instances. It is intrusive because every time a
class definition is used instead of a differing data, the amount of code
required to utilise that specific entity type in a given circumstance
increases. In many codebases there is the concept of the player, and the
player has different attributes to other entities, such as lacking AI
controls, or having player controls, or having regenerating health, or
having ammo. All these different traits can be inferred from data
decisions, but sometimes they are made literal through code in classes.
When these differences are put in code, interfacing between different
classes becomes a game of adapting, a known design pattern that should
be seen as a symptom of mixed levels of specialisation in a set of
classes. When developing a game, this ususally manifests as time spent
writing out templated code that can operate on multiple classes rather
than refactoring the classes involved into more discrete components.
This could be considered wasted time as the likelyhood of other
operations needing to operate on all the objects is greater than zero,
and the effort to refactor into components is usually no greater than
the effort to create a working templated operation.

### Domain Key / Knowledge

Whereas first normal form is almost a mechanical process, 2nd Normal
form and beyond become a little more investigative. It is required that
the person doing the normalisation understand the data in order to
determine whether parts of the data are dependent on the key and whether
or not they would be better suited in a new table, or written out as a
form of procedural result due to domain knowledge.

Domain key normal form is normally thought of as the last normal form,
but for developing efficient data structures, it’s one of the things
that is best studied early and often. The term domain knowledge is
preferrable when writing code as it makes more immediate sense and
encourages use outside of keys and tables. Domain knowledge is the idea
that data depends on other data, given information about the domain in
which it resides. Domain knowledge can be as simple as knowing the
collowquialism for something, such as knowing that a certain number of
degrees Celsius or Fahrenheit is hot, or whether some SI unit relates to
a man-made concept such as 100m/s being rather quick. These procedural
solutions are present in some operating systems or applications: the
progress dialogue that may say “about a minute” rather than an
innaccurate and erratic seconds countdown. Hoever, domain knowledge
isn’t just about human interpretation of data. For example things such
as boiling point, the speed of sound, of light, speed limits and average
speed of traffic on a given road network, psychoacoustic properties, the
boiling point of water, and how long it takes a human to react to any
given visual input. All these facts may be useful in some way, but can
only be put into an application if the programmer adds it specifically
as procedural domain knowledge or as an attribute of a specific
instance.

Domain knowledge is useful because it allows us to lose some otherwise
unnecessarily stored data. It is a compiler’s job to analyse the
produced output of code (the abstract syntax tree) to then provide
itself with data upon which it can infer and use domain knowledge about
what operations can be omitted, reordered, or transformed to produce
faster or cheaper assembly. Profile guided optimisation is another way
of using domain knowledge, the domain being the runtime, the knowledge
being the statistics of instructino coverage and order of calling. PGO
is used to tune the location of instructions and hint branch predictors
so that they produce even better performance based on a real world use
case.

### All Normal Forms

First normal form: remove repeating columns and nulls by adding new
tables for optional values or one to may relationships. That is, ensure
that any potentially null columns in your table are moved to their own
table and use the primary table’s key as their primary key. Move any
repeating columns (such as item1, item2, item3) to a separate single
table (item) and again use the original table’s primary key as the
primary key for this new table.

Second, Third, and BC (Boyce-Codd) normal form: split tables such that
changes to values only affect the minimum amount of tables. Make sure
that columns rely on only the table key, and also all of the table key.
Look at all the columns and be sure that each one relies on all of the
key and not on any other column.

Fourth normal form: ensure that your tables columns are truly dependent
on each other. An example might be keeping a table of potential colours
and sizes of clothing, with an entry for each colour and size
combination. Most clothes come in a set of sizes, and a set of colours,
and don’t restrict certain colours based on size so there would be no
omissions from a matrix of all the known sizes and all the known
colours, instead the table should be two tables, both with the garment
as primary key, with size or colour as the data. This holds for
specifications, but doesn’t hold for the case where an item needs to be
checked for whether or not it is in stock. Not all sizes and colour
combinations may be in stock, so an “in-stock” cache table would be
right to have all three columns.

Fifth normal form: if the values in multiple columns are related by
possibility, then remove the columns out into the separate tables of
possible combinations. For example, an Ork can use simple weapons or
orcish weapons, a human can use simple weapons or human weapons, if we
say that Human Arthur is using a stick, then the stick doesn’t need the
column specifying that it is an simple weapon, but equally, if we say
the Ork KruelGut is using a sword, we don’t need a column specifying
that he is using an orcish sword, as he cannot use a human sword and
that is the only other form of sword available. In this case then we
have a table that tells us what each entity is using for a weapon (we
know what race each entity is in another table), and we have a table for
what weapon types are available per race, and the weapon table can still
have both weapontype and actual weapon. The benefit of this is that the
entities only need to maintain a lower range “weapon” and not also the
weapontype as the weapontype can be inferred from the weapontypes
available for the race. This reduction in memory useage may come at a
cost of performance, or may increase performance depending on how the
data is accessed.

Sixth normal form: This is an uncommon normal form because it is one
that is normally only important for keeping track of changing states and
can cause an explosion in the number of tables. However, it can be
useful in reducing memory usage when tracking rapidly changing
independent columns of an otherwise fully normalised table. For example,
if a character is otherwise fully normalised, but they need to keep
track of how much money, XP, and spell points they have over time, it
might be better to split out the money, XP, and spell points columns
into character/time/value tables that can be separately updated without
causing a whole new character record to be written purely for the sake
of time stamping.

Domain/Key normal form: using domain knowledge remove columns or parts
of column data because it depends on some other colum through some
intrinsic quality of the domain in which the table resides. Information
provided by the data may already be available in other data. For eaxmple
the colloquialism of old-man is dependent on having data on the gender
column and age colum, and can be inferred. Thus it doesn’t need to be in
a column and instead can live as a process. Equally, foreshadowing
existence based processing here, a dead ork has zero health and a unhurt
ork has zero damage. If we maintain a table for partially damaged orks
health values then we don’t need storage space undamaged ork health.
This minor saving can add up to large savings when operating on multiple
entities with multiple stats that can vary from a known default.

### Sum up

At this point we can see it is perfectly reasonable to store any highly
complex data structures in a database format, even game data with its
high interconnectedness and rapid design changing criteria. What is
still of concern is the less logical and more binary forms of data such
as material data, mesh data, audio clips and runtime integration with
scripting systems and control flow.

Platform specific resources such as audio, texture, vertex or video
formats are opaque to most developers, and moving towards a table
oriented approach isn’t going to change that. In databases, it’s common
to find column types of boolean, number, and string, and when building a
database that can handle all the natural atomic elements of our engine,
it’s reasonable to assume that we can add new column types for these
platform or engine intrinsics. The textureID and meshID column used in
room examples could be smart pointers to resources. Creating new meshes
and textures may be as simple as inserting a new row with the asset URL
and keeping track of whether or not an entry has arrived in the
fulfilled streaming request table or the URL to assetID table that could
be populated by a behind the scenes task scheduler.

As for integration with scripting systems and using tables for logic and
control flow, chapters on finite state machines, existence based
processing, condition tables and hierarchical level of detail show how
tables don’t complicate, but instead provide opportunity to do more with
fewer resources as results flow without constraint normally associated
with object or entity linked data structures.

Implicit Entities
-----------------

Getting your data out of a database and into your objects can appear
quite daunting, but a database approach to data storage does provide a
way to allow old executables to run off new data, it also allows new
executables to run off old data, which is can be vital when working with
other people who might need to run an earlier or later version. We saw
that sometimes adding new features required nothing more than adding a
new table. That’s a non-intrusive modification if you are using a
database, but a significant change if you’re adding a new member to a
class. If you’re trying to keep your internal code object-oriented, then
loading up tables to generate your objects isn’t as easy as having your
instances generated from script calls, or loading them in a binary file
and doing some pointer fixup. Saving can be even more of a nightmare as
when going from objects back to rows, it’s hard to tell if the database
needs rows added, deleted, or just modified. This is the same set of
issues we normally come across when using source control. The problem
with figuring out what was moved, deleted, or added when doing a three
way merge.

But again, this is only true if you convert your database formatted
files into explicit objects. These denormalised objects are troublesome
because they are in some sense, self aware. They are aware of what their
type is, and what to a large degree, what their type could be. A problem
seen many times in development is where a base type needs to change
because a new requirement does not fit with the original vision.
Explicit objects act like the original tables before normalisation: they
have a lot of nulls when features are not used. If you have a reasonable
amount of object instances, this wasted space can cause problems.

You can see how an originally small object can grow to a
disproportionate size, how a Car class can gain 8 wheel pointers in case
it is a truck, how it can gain up to 15 passengers, optionally having
lights, doors, sunroof, collision mesh, different engine, a driver base
class pointer, 2 trailers, and maybe a physics model base class pointer
so you can switch to different physics models when level collision is or
isn’t present. All these are small additions, not considered bad in
themselves, but every time you check a pointer for null it can turn out
either way. If you guess wrong, or the branch predictor guesses wrong,
the best you can hope for is an instruction cache miss or pipeline flush
before doing an action. Pipeline flushes may not be very expensive on
out-of-order CPUs, but the PS3 and Xbox360 have in-order CPUs and suffer
a stall when this happens.

It is generally assumed that if you bind your type into an explicit
object, you also allow for runtime polymorphism. This is seen as a good
thing, in that you can keep the over arching game code smaller and
easier to understand. You can have a main loop that runs over all of
your objects, calling: “Think”, “Update”, “Render”, and then you loop
forever. But runtime polymorphism is not only made possible by using
explicit types in an Object-oriented system. In the set of tables we
just normalised, the Pickups were optionally coloured. In traditional
Object-oriented C++ games development we generally define the Pickups,
and then either have an optional component for tinting or derive from
the Pickup type and add more information to the GetColour member
function. In either case it can be necessary to override the base class
in some fashion. You can either add the optional tint explicitly or make
a new class and change the Pickup factory to accept that some Pickups
can have colour tinting. In our database normalisation, adding tinting
required only adding a new table and populating it with only the Pickups
that were tinted.

The more commonly seen as a bad approach (littering with pointers to
optional things) is problematic in that it leads to a lot of cases where
you cannot quickly find out whether an object needs special care unless
you check this extended type info. For example, when rendering the
Pickups, for each one in every frame, we might need to find out whether
to render it in the alpha blend pass or in the solid pass. This can be
an important question, and because it is asked every frame in most
cases, it can add some dark matter to our profile, some cold code
ineffciency.

The more commonly seen as correct approach (that is create a new type
and adjust the factory), has a limitation that you cannot change a
Pickup from non-tinted to tinted at runtime without some major changes
to how you use Pickups in the code. This means that if an object is
tinted, it remains tinted unless you have the ability to clone and swap
in the new object. This seems quite strict in comparison to the pointer
littering technique. You may find that you end up making all pickups
tinted in the end, just because they might be tinted at some time in the
future. This would mean that you would have the old code for handling
the untinted pickup rotting while you assume it still works. This has
been the kind of code that causes one in a thousand errors that are very
hard to debug unless you get lucky.

The database approach maintains the runtime dynamicity of the pointer
littering approach by allowing for the creation of tint rows at runtime
post creation. It also maintains the non-littering aspect of the derived
type approach because we didn’t add anything to any base type to add new
functionality. It’s quicker than both for iterating, which in our
example was binning into alpha blend render pass and solid render pass.
By only iterating over the table of tints picking out which have alpha
values that cause it to be binned in alpha blended, we save memory
accesses as we’re only traversing the list of potentially alpha blended
rather than running over all the objects. All objects not in the
TintedPickup set but in the Pickup set can be assumed to be solid
rendered, no checking of anything required. We have one more benefit
with the separation of data in that it is easier to prefetch rows for
analysis when they are much simpler in layout, and we could likely have
more rows in the cache at once than in either of the other methods.

The fundamental difference between the database style approach and the
object-oriented approach is how the type of a Pickup is defined.
Traditionally, we derive new explicit types to allow new functionality
or behaviours. With databases, we add new tables and into them insert
rows referencing existing entities to show new information about old
information. We imply new functionality while not explicitly adding
anything to the original form.

In an explicit entity system, the noun is central, the entity has to be
extended with data and functions. In an implicit entity system, the
adjectives are central, they refer to an entity, implying its existence
by recognising it as being their operand. The entity only exists when
there is something to say about it. By contrast, in an explicit entity
system, you can only say something about an entity that already exists
and caters for that describability.

Components imply entities
-------------------------

If we go back to our level file, we see that the room table is quite
explicit about itself. What we have is a simple list of rooms with all
the attributes that are connected to the room written out along the
columns. Although adding new features is simple enough, modification or
reusing any of these separately would prove tricky.

If we change the room table into multiple adjective tables, we can see
how it is possible to build it out of components without having a
literal core room. We will add tables for renderable, position, and exit

[h]Room Renderable\

<span>lllll</span> **<span>RoomID</span>&MeshID&TextureID\
r1&m1&t1\
r2&m2&t2\
r3&m3&t3\
**

\
Room Position\

<span>lllll</span> **<span>RoomID</span>&WorldPos\
r1&0,0\
r2&20,5\
r3&30,-5\
**

\
Exit Room\

<span>lllll</span> **<span>RoomID</span>\
r3\
**

\

With the new layout there is no core room. It is only implied through
the fact that it is being rendered, and that it has a position. In fact,
the last table has taken advantage of the fact that the room is now
implicit, and changed from a bool representing whether the room is an
exit room or not into a one column table. The entry in that table
implies the room is the exit room. This will give us an immediate memory
usage balance of one bool per room compared to one row per exit room.
Also, it becomes slightly easier to calculate if the player is in an
exit room, they simply check for that room’s existence in the ExitRoom
table and nothing more.

Another benefit of implicit entities is the non-intrusive linking of
existing entities. In our data file, there is no space for multiple
doors per room. With the pointer littering technique, having multiple
doors would require an array of doors, or maybe a linked list of doors,
but with the database style table of doors, we can add more rows
whenever a room needs a new door.

[h]Doors\

<span>ll</span> **<span>RoomID</span>&bf<span>RoomID</span>\
r1&r2\
r2&r3\
**

\
Locked Doors\

<span>lll</span> **<span>RoomID</span>&bf<span>RoomID</span>&Locked\
r2&r3&k1\
**

\

Notice that we have to make the primary key the combination of the two
rooms. If we had kept the single room ID as a primary key, then we could
only have one row per room. In this case we have allowed for only one
door per combination of rooms. We can guarantee in our game that a room
will only have one door that leads to another room; no double doors into
other rooms. Because of that, the locks also have to switch to a
compound key to reference locked doors. All this is good because it
means the system is extending, but by changing very little. If we
decided that we did need multiple doors from one room to another we
could extend it thus:

[h]Doors\

<span>lll</span> **<span>DoorID</span>&RoomID&RoomID\
d1&r1&r2\
d2&r2&r3\
**

\
Locked Doors\

<span>ll</span> **<span>DoorID</span>&Locked\
d2&k1\
**

\

There is one sticking point here, first normal form dictates that we
should build the table of Doors differently, where we have two columns
of doors, it requires that we have one, but multiple rows per door.

[h]Doors\

<span>ll</span> **<span>DoorID</span>&RoomID\
d1&r1\
d1&r2\
d2&r2\
d2&r3\
**

\
Locked Doors\

<span>ll</span> **<span>DoorID</span>&Locked\
d2&k1\
**

\

we can choose to use this or ignore it, but only because we know that
doors have two and only two rooms. A good reason to ignore it could be
that we assume the door opens into the first door. Thus these two room
IDs actually need to be given slightly different names, such as
*DoorInto*, and *DoorFrom*.

Sometimes, like with RoomPickups, the rows in a table can be thought of
as instances of objects. In RoomPickups, the rows represent an instance
of a PickupType in a particular Room. This is a many to many
relationship, and it can be useful in various places, even when the two
types are the same, such as in the RoomDoors table.

When most programmers begin building an entity system they start by
creating an entity type and an entity manager. The entity type contains
information on what components are connected to it, and this implies a
core entity that exists beyond what the components say about it. Adding
information about what components an entity has directly into the core
entity might help debugging for a little while, while the components are
all still centred about entities, but it becomes cumbersome when we
realise the potential of splitting and merging entities and have to move
away from using an explicit entity inspector. Entities as a replacement
for classes, have their place, and they can simplify the move from class
central thinking to a more data-oriented approach.

Cosmic Hierarchies
------------------

Whatever you call them, be it Cosmic Base Class, Root of all Evil,
Gotcha \#97, or CObject, having a base class that everything derives
from has pretty much been a universal failure point in large C++
projects. The language does not naturally support introspection or duck
typing, so it has difficultly utilising CObjects effectively. If we have
a database driven approach, the idea of a cosmic base class might make a
subtle entrance right at the beginning by appearing as the entity to
which all other components are adjectives about, thus not letting
anything be anything other than an entity. Although component–based
engines can often be found sporting an EntityID as their owner, not all
require owners. Not all have only one owner. When you normalise
databases, you find that you have a collection of different entity
types. In our level file example we saw how the objects we started with
turned into a MeshID, TextureID, RoomID, and a PickupID. We even saw the
emergence through necessity of a DoorID. If we pile all these Ids into a
central EntityID, the system should work fine, but it’s not a necessary
step. A lot of entity systems do take this approach, but as is the case
with most movements, the first swing away swings too far. The balance is
to be found in practical examples of data normalisation provided by the
database industry.

Structs of Arrays
-----------------

In addition to all the other benefits of keeping your runtime data in a
database style format, there is the opportunity to take advantage of
structures of arrays rather than arrays of structures. SoA has been
coined as a term to describe an access pattern for object data. It is
okay to keep hot and cold data side by side in an SoA object as data is
pulled into the cache by necessity rather than by accidental physical
location.

If your animation timekey/value class resembles this:

    struct Keyframe
    {
        float time, x,y,z;
    };
    struct Stream
    {
        Keyframe *keyframes;
    };

[src:animstruct]

then when you iterate over a large collection of them, all the data has
to be pulled into the cache at once. If we assume that a cacheline is
128 bytes, and the size of floats is 4 bytes, the Keyframe struct is 16
bytes. This means that every time you look up a key time, you
accidentally pull in four keys and all the associated keyframe data. If
you are doing a binary search of a 128 key stream, that could mean you
end up loading 128bytes of data and only using 4 bytes of it in up to 6
chops. If you change the data layout so that the searching takes place
in one array, and the data is stored separately, then you get structures
that look like this:

~~~~ {caption="struct" of="" arrays=""}
struct KeyData
{
    float x,y,z;
};
struct stream
{
    float *times;
    KeyData *values;
};
~~~~

[src:SoAclass]

Doing this means that for a 128 key stream, a binary search is going to
pull in at most three out of four cachelines, and the data lookup is
guaranteed to only require one.

Database technology also saw this, it’s called column oriented databases
and the provide better throughput for data processing over traditional
row oriented relational databases simply because irrelevant data is not
loaded when doing column aggregations or filtering.

Stream Processing
-----------------

Now we realise that all the game data and game runtime can be
implemented in a database oriented approach, there’s one more
interesting side effect: data as streams. Our persistent storage is a
database, our runtime data is in the same format as it was on disk, what
do we benefit from this? Databases can be thought of as collections of
rows, or collections of columns, but there is one more way to think
about the tables, they are sets. The set is the set of all possible
permutations of the attributes. For example, in the case of RoomPickups,
the table is defined as the set of all possible combinations of Rooms
and Pickups. If a room has a Pickup, then the bit is set for that
room,pickup combination. If you know that there are only N rooms and M
types of Pickup, then the set can be defined as a bit field as that is
N\*M bits long. For most applications, using a bitfield to represent a
table would be wasteful, as the set size quickly grows out of scope of
any hardware, but it can be interesting to note what this means from a
processing point of view. Processing a set, transforming it into another
set, can be thought of as traversing the set and producing the output
set, but the interesting attribute of a set is that it is unordered. An
unordered list can be trivially parallel processed. There are massive
benefits to be had by taking advantage of this trivialisation of
parallelism wherever possible, and we normally cannot get near this
because of the data layout of the object-oriented approaches.

Coming at this from another angle, graphics cards vendors have been
pushing in this direction for many years, and we now need to think in
this way for game logic too. We can process lots of data quickly as long
as we utilise about stream processing as much as possible and use random
access processing as little as possible. Stream processing in this case
means to process data without having variables that are external to the
current datum., thus ensuring the processes or transforms are trivially
parallelisable.

When you prepare a primitive render for a graphics card, you set up
constants such as the transform matrix, the texture binding, any
lighting values, or which shader you want to run. When you come to run
the shader, each vertex and pixel may have its own scratch pad of local
variables, but they never write to globals or refer to a global
scratchpad. Enforcing this, we can guarantee parallelism because the
order of operations are ensured to be irrelevant. If a shader was
allowed to write to globals, there would be locking, or it would become
an inherently serial operation. Neither of these are good for massive
core count devices like graphics cards, so that has been a self imposed
limit and an important factor in their design.

Doing all processing this way, without globals / global scratchpads,
gives you the rigidity of intention to highly parallelise your
processing and make it easier to think about the system, inspect it,
debug it, and extend it or interrupt it to hook in new features. If you
know the order doesn’t matter, it’s very easy to rerun any tests or
transforms that have caused bad state.

Beautiful Homogeneity
---------------------

Apart from all the speed increases and the simplicity of extension,
there is also an implicit tendency to turn out accidentally reusable
solutions to problems. This is caused by the data being formatted much
more rigidly, and therefore when it fits, can almost be seen as a type
of duck-typing. If the data can fit a transform, a transform can act on
it. Some would argue that just because the types match, doesn’t mean the
function will create the expected outcome, but this is simply avoidable
by not reusing code you don’t understand. Because the code becomes much
more penetrable, it takes less time to look at what a transform is doing
before committing to reusing it in your own code.

Another benefit that comes from the data being built in the same way
each time, handled with transforms and always being held in the same
types of container is that there is a very good chance that there are
multiple intention agnostic optimisations that can be applied to every
part of the code. General purpose sorting, counting, searches and
spatial awareness systems can be attached to new data without calling
for OOP adapters or implementing interfaces so that Strategies can run
over them.

A final reason to work with data in an immutable way comes in the form
of preparations for optimisation. C++, as a language, provides a lot of
ways for the programmer to shoot themselves in the foot, and one of the
best is that pointers to memory can cause unexpected side effects when
used without caution. Consider this piece of code:

~~~~ {caption="byte" copying=""}
char buffer[ 100 ];
buffer[0] = 'X';
memcpy( buffer+1, buffer, 98 );
buffer[ 99 ] = '\0';
~~~~

this is perfectly correct code if you just want to get a string with 99
’X’s in it. However, because this is possible, memcpy has to copy one
byte at a time. To speed up copying, you normally load in a lot of
memory locations at once, then save them out once they are all in the
cache. If your input data can be modified by your output buffer, then
you have to tread very carefully. Now consider this:

~~~~ {caption="trivially" parallelisable="" code=""}
int q=10;
int p[10];
for( int i = 0; i < q; ++i )
    p[i] = i;
~~~~

The compiler can figure out that q is unaffected, and can happily unroll
this loop or replace the check against q with a register value. However,
looking at this code instead:

~~~~ {caption="potentially" aliased="" int=""}
void foo( int* p, const int &q )
{
    for( int i = 0; i < q; ++i)
        p[i] = i;
}

int q=10;
int p[10];
foo( p, q );
~~~~

The compiler cannot tell that q is unaffected by operations on p, so it
has to store p and reload q every time it checks the end of the loop.
This is called aliasing, where the address of two variables that are in
use are not known to be different, so to ensure correctly functioning
code, the variables have to be handled as if they might be at the same
address.

[^1]: There are existing APIs that present strings in various ways
    dependent on how they are used, for example the difference between
    human readable strings (usually UTF-8) and ascii strings for
    debugging. Adding sounds, textures, and meshes to this seems quite
    natural once you realise that all these things are resources for
    human consumption

[^2]: in fact, the filename is the primary key of the filedata on the
    storage media, filesystems are simple key-value databases with very
    large complex/large values