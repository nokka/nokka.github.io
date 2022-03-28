---
title: "Writing a Diablo II binary parser using Go"
date: "2022-02-27"
tags: ["diablo-ii"]
draft: false
---

In 2016 I joined a private Diablo II server called Slashdiablo. It's run by a few Diablo enthusiasts and
alot of code and administration tools had been written to keep the place running. As I started playing and engaging
with the community on Discord I quickly noticed user interfacing services I was missing from modern 
games such as an easy way to see player rankings or the ability to check character builds online.

I started to dig into the architecture of Diablo II and how they were saving the character data I was 
interested in. It turns out there were some people back in the day (early 2000's) who spent a lot of time
reverse engineering the game, but since then a lot of that work had been lost. To be fair it's been almost
20 years since the work was done, but I did find some information on how to proceed and piece a lot of data
together and spent a lot of time testing, writing tools to try different bit lengths and reverse engineer data. 
This write up will focus on how to parse binary data using Go, I might do an attempt to write about
the research process down the line.

It turns out Diablo II stores all character data in tightly packed binary files called `.d2s` files.
This binary layout was designed when every bit mattered so they didn't waste a single bit packing data.

![D2s binary layout](/images/binary.png)

The image above shows the layout of the character binary byte after byte. As it turns out the file
has seven sections delimited by `2 byte` header string, always containing a 2 character string.

Representing this character as a Go struct without the header strings would look like this.

```go
type Character struct {
	Header      Header     
	Attributes  Attributes 
	Skills      []Skill    
	Items       []Item     
	CorpseItems []Item     
	MercItems   []Item     
	GolemItem   *Item // Optional item, only for Necromancers.
}
```

The first three sections of the binary has a fixed byte length, this was important because at this point I had no idea
what I was doing and having a fixed length reduced the complexity.

So I decided to read the first `765 byte` header section, but I also knew I had to keep reading
the other sections afterwards. This means I need to keep a pointer to which byte in the `.d2s` file I'm currently
reading at. Luckily Go has this really nifty package called `bufio`. This package exports a
buffered reader that wraps the `io.Reader` interface. A buffered reader basically means we're able to read
parts of an input stream and then continue from that point afterwards, since the `bufio.Reader` keeps a pointer to the
last byte read it's a perfect fit.


```go
// Read the character binary from disk. io.File implements io.Reader.
file, err := os.Open("/Users/stekon/go/src/github.com/nokka/d2s-article/nokkasorc")
if err != nil {
    return err
}

// Create a buffered reader using the file content.
bfr := bufio.NewReader(file)

// Create a byte slice of exactly 765 bytes that can hold the header data.
buf := make([]byte, 765)

// Read the full length of buf length into the buf byte slice.
_, err = io.ReadFull(bfr, buf)
if err != nil {
    return err
}

fmt.Println(buf)
```

When we print the content of the `buf` byte slice as a string, we can see that it contains just a bunch
of binary gibberish, but there's also the text `Woo!` from which we can conclude that the Diablo II 
developers had a sense of humor.

![header binary string](/images/buf_string.png)

This binary string is no good use for us as is, but from the previously mentioned research, I knew the data
available from the header and their byte lengths. This means that we can use a `struct` in Go with the correct
byte lengths to slot the bytes into place. Below is a struct with the first ten data points in the header.
The rest is omitted since the header is really long, but you can find the full struct [here](https://github.com/nokka/d2s/blob/master/character.go#L90).
```go
// Small sample of the first fields of the header.
type Header struct {
	Identifier  uint32    // 4 bytes
	Version     uint32    // 4 bytes
	FileSize    uint32    // 4 bytes
	CheckSum    uint32    // 4 bytes
	ActiveArms  uint32    // 4 bytes
	Name        [16]byte  // 16 bytes
	Status      byte      // 1 byte
	Progression byte      // 1 byte
	_           [2]byte   // 2 byte, Unknown field
	Class       byte      // 1 byte
	Omitted     [724]byte // 724 omitted bytes
}
```

Now that we have a struct to read the buffer into, we can use the `binary.Read` func
from the Go standard library. It takes three arguments.

- The first one is an `io.Reader` that we pass our `buf` to
- The second argument is the [byte order](https://en.wikipedia.org/wiki/Endianness),
which determines the order the bytes are sequenced in, also called Endianness and d2s files are in 
the order Little Endian.
- The last argument is the struct we want to read the data into (hence the pointer), and this our header struct.

```go
// Create a new character.
var character Character

// Read the content of the buffer into the header data struct.
err = binary.Read(bytes.NewBuffer(buf), binary.LittleEndian, &character.Header)
if err != nil {
    return err
}

fmt.Println(character.Header.Identifier)
fmt.Println(character.Header.CheckSum)
fmt.Println(strings.Trim(string(character.Header.Name[:]), "\x00"))
```

Ok so now that we have our header data, lets confirm that we have some useful stuff in there,
instead of this binary gibberish. Let's look at the name because it's a bit tricky. It's a `[16]byte` to fit
all potential names allowed in game. Our character name is a bit shorter, which means there's a few
empty bytes at the end.

```go
[78 111 107 107 97 83 111 114 99 0 0 0 0 0 0 0]
```

To get rid of these we can simply trim all the trailing empty characters `\x00`. 
Running the program right now will output the identifier, the checksum and the character name.

```bash
$ go run main.go
1437226410
3173499472
NokkaSorc
```

## Attributes
The attributes section while having a fixed length do introduce some interesting complexity.
The attributes layout consists of a list of attributes such as strength, dexterity, health or mana.
Each attribute starts with a `9 bit` ID that will be immediately followed by an `n` bit value.
The list is finally terminated by a `9 bit` `0x1ff` value. Every attribute value has a different bit length
and the mapping can be found [here](https://github.com/nokka/d2s/blob/master/attributes.go#L44-L61).

This is interesting because the data is no longer byte aligned. Simply put this means data can span several 
bytes or stop in the middle of a byte. To read `n` bits at a time we need to introduce a bit reader. There are
several options for this in Go, most of them read bits in a Big Endian order but Diablo II stores the data
in Little Endian. This means I had to rely on my own implementation of a bit reader which you can find
[here](https://github.com/nokka/d2s/blob/master/bitreader.go).

After the attribute ID has been determined we can switch on the IDs and slot the data into the correct place
on the Attributes struct. Some values are normalized and divided by 256 before they are stored in the struct.

```go
// Create a bit reader from the same buffered reader to read the stats.
br := bitReader{r: bfr}

// Read the 2 header bytes and make sure we're correctly aligned.
g, err := br.ReadByte()
if err != nil {
	return err
}

f, err := br.ReadByte()
if err != nil {
	return err
}

if string(g) != "g" || string(f) != "f" {
	return errors.New("failed to find attributes header gf")
}

for {
	id, err := br.ReadBits(9)
	if err != nil {
		return err
	}

	// If all 9 bits are set, we've hit the end of the attributes section
	//  at 0x1ff and exit the loop.
	if id == 0x1ff {
		break
	}

	// The attribute value bit length, so we'll know how many bits to read next.
	length, ok := attributeBitMap[id]
	if !ok {
		return fmt.Errorf("unknown attribute id: %d", id)
	}

	// The attribute value.
	attr, err := br.ReadBits(length)
	if err != nil {
		return err
	}

	switch id {
	case strength:
		character.Attributes.Strength = attr
	case energy:
		character.Attributes.Energy = attr
	case dexterity:
		character.Attributes.Dexterity = attr
	case vitality:
		character.Attributes.Vitality = attr
	case unusedStats:
		character.Attributes.UnusedStats = attr
	case unusedSkills:
		character.Attributes.UnusedSkillPoints = attr
	case currentHP:
		character.Attributes.CurrentHP = attr / 256
	case maxHP:
		character.Attributes.MaxHP = attr / 256
	case currentMana:
		character.Attributes.CurrentMana = attr / 256
	case maxMana:
		character.Attributes.MaxMana = attr / 256
	case currentStamina:
		character.Attributes.CurrentStamina = attr / 256
	case maxStamina:
		character.Attributes.MaxStamina = attr / 256
	case level:
		character.Attributes.Level = attr
	case experience:
		character.Attributes.Experience = attr
	case gold:
		character.Attributes.Gold = attr
	case stashedGold:
		character.Attributes.StashedGold = attr
	}
}
```

## Skills
As we've seen so far each section starts with a `2 byte` header, it's also a good way for us to be sure
that we have read the correct amount of bits on the attributes section and this is indeed the start of the 
skills section. After the `2 byte` header there's a fixed length of `[30]byte` that contains allocated skill points.
We can read them the exact same way we did the header data, since the bufio reader is queued at the correct
byte already we can simply read the 32 bytes into a struct.

```go
// Define a struct that can hold all skill data.
type skillData struct {
	Header [2]byte
	List   [30]byte
}

// Make a buffer that can hold 32 bytes, which can hold the entire skillset.
buf := make([]byte, 32)

_, err := io.ReadFull(bfr, buf)
if err != nil {
	return err
}

skillHeaderData := skillData{}
err := binary.Read(bytes.NewBuffer(buf), binary.LittleEndian, &skillHeaderData)
if err != nil {
	return err
}
```

Now begins an interesting part of how the skills in Diablo II are layed out.
Basically all 357 skills used by players, monsters and bosses are in a list.
Each unique class or monster has an offset associated with them, and this offset
is used to read the skills of the particular actor. Each playable class has
exactly 30 skills available to them and the skill list is `30 byte` long. Each
byte contains an integer with the number of allocated points in that specific skill.
As long as we know the offset for the class of the binary we are reading and
know the class ID (available in header), we can simply iterate the list of integers
and slot them into the correct skill ID starting at the offset and iterating 30 times.


The skill map is omitted in the example but the full list can be found [here](https://github.com/nokka/d2s/blob/master/skills.go#L20-L378).

```go
// Skill represents an available character skill in Diablo II.
type Skill struct {
	ID     int    
	Points int    
	Name   string
}

// Skill offsets for each class ID. For example if the character
// is a Paladin, start reading skills at index 96.
var skillOffsetMap = map[uint]int{
	Amazon:      6,
	Sorceress:   36,
	Necromancer: 66,
	Paladin:     96,
	Barbarian:   126,
	Druid:       221,
	Assassin:    251,
}

if string(skillHeaderData.Header[:]) != "if" {
	return fmt.Errorf("failed to find skill header")
}

// Find the skill offset for the character class using the offset map.
skillOffset, ok := skillOffsetMap[uint(character.Header.Class)]
if !ok {
	return fmt.Errorf("unknown skill offset for class %d", character.Header.Class)
}

// Loop through the list of allocated points and slot the skills into place,
// since they are in order we can simply use iterator + the offset of the skill map.
for i, allocatedPoints := range skillHeaderData.List {
	id := (i + skillOffset)

	skillName, ok := skillMap[id]
	if !ok {
		return fmt.Errorf("unknown skill id %d", id)
	}
	s := Skill{
		ID:     id,
		Points: int(allocatedPoints),
		Name:   skillName,
	}
	character.Skills = append(character.Skills, s)
}
```

## Item list
This is where it gets seriously complicated and I spent probably around 90% of the time on this part.
If you are familiar with Diablo II you know there's a wide array of different type of items, rarities,
prefixes, suffixes and magical properties. Depending on the item rarity it can have different type of
properties such as unique items having set number of magical attributes, or rare and crafted items having two rare
names put together randomly that determines the magical attributes they will obtain, for example `Corruption Grip Ring`.

Each item also has an item level and this item level determines which magical attributes are eligible, this means certain
magical attributes can only occur above certain item levels.

Instead of trying to link the thousands of lines of code that perform the complex task of parsing these items I will try and explain the differences between item types and give an overview of how the parsing works on the binary level, and then link to the full implementation of the item parser in the library.

The item list as all the other sections start with a `2 byte` header containing the string `JM` but the item list
also have an uint16 (2 byte) after the header containing information about the number of items to read in the list.
This is important because we don't know how long this section is until we have read it. Each item has a bit length of `n` depending on rarity and item type. For example not all items have durability or defense.


All items have a quality property that determines the rarity of the item, the higher the value the more complex the item is.

```go
// Rarity IDs.
const (
	lowQuality        = 1
	normal            = 2
	highQuality       = 3
	magicallyEnhanced = 4
	partOfSet         = 5
	rare              = 6
	unique            = 7
	crafted           = 8
)
```

### Simple items
Simple items are always `111 bit` long they contain the most basic data that all items have, like if it's ethereal, how
many number of sockets it has, the name of the item and its position in the inventory or slot it is equipped in. All items have this information so consider it the "base" of each item.

### Magical items
A magical item has both an affix and a suffix, for example `Russet Amulet of Luck`, where `Russet` is the prefix and `Luck` is the suffix. These affixes and suffixes can roll on items based on their item level and have a range as well.

```
INSERT IMAGE
```

All items above rarity ID > 4 have this list of magical properties assigned to them. This list is similar to the attributes section where the magical list looks like this. 

```
for {
	9 bit id
	n bits of magical properties
	0x1ff terminator
	When we hit the terminator 0x1ff the next item begins.
}
```

The full magical list reader can be found [here](https://github.com/nokka/d2s/blob/426ae713940b7474a5f7872f16dddb02ced8a241/d2s.go#L991-L1034).

The tricky part about reading these magical properties is that can be quite different, they are `n` bits long but these
bits an be divided into several sections depending on the property. Representing them in Go I use a `[]uint` to divide to help with understanding how long each section will be.

```go
type MagicalProperty struct {
	Bits []uint // slice of ints representing number of bits to read.
	Bias uint64 // bias is a value to substract from a given property if needed
	Name string // name of the magical property
}
```

A simple magical property like strength with ID `0` that only has one section of bits looks like this.

```go
var magicalProperties = map[uint64]MagicalProperty{
	0:  {Bits: []uint{8}, Bias: 32, Name: "+{0} to Strength"},
	...
}
```

A more complicated magical property with ID `83` that has several sections of data is for example the + Skills property.
It has two sections each `3` bits each.
```go
var magicalProperties = map[uint64]MagicalProperty{
	...
	83: {Bits: []uint{3, 3}, Name: "+{1} to {0} Skill Levels"},
	...
}
```

### Rare & crafted items
Rare items 



---

This is by far the most tricky part to read. The items section starts of with a 4 byte header, containing the value JM, and a uint16 value which is the item count your character currently has. Equipped, inventory, stash, cube and belt all included.

The byte length of the section is unknown before reading it in it's entirety, because the bit length of each item varies depending on its quality, number of sockets and magical attributes it possess.

Each item follows a certain pattern though, which is:

Simple items
Each item starts of with 111 bits of simple data, which all items contain. This is information like item type, if it's socketed, position id like equipped or stash and so on.

Each item also has a boolean called SimpleItem which is 1 bit long, if this is set to 1, the item contains no more bits, and the next item starts.

Advanced items
If the item is not a simple items, this means it will have tons of data following the initial 111 bits. A few examples of this is the rarity level, magical suffix, magical affix, if it's a runeword, personalized, part of a set, class specific and so on.

Last but not least if the item has will have lists of magical properties depending on if it's a runeword, magical, rare, crafted, unique part of set and so on.

These lists are similar to the attributes section where we will read:

9 bit id
n bits of magical properties
0x1ff terminator
When we hit the terminator 0x1ff the next item begins.

Magical properties
A magical property is a unique property that can occur on an item, each property has a different bit length, and the map is huge.

Example
This is the magical property with id 83 which contains 2 bit fields each 3 bits long.

83: {Bits: []uint{3, 3}, Name: "+{1} to {0} Skill Levels"},
All magical properties are mapped in the item.go file.