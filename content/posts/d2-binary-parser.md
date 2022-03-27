---
title: "Writing a Diablo II binary parser using Go"
date: "2022-02-27"
tags: ["diablo-ii"]
draft: false
---

In 2016 I joined a private Diablo II server called Slashdiablo. It's run by a few Diablo enthusiasts and
alot of code and tools had been written to keep the place running. As I started playing and engaging
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
has seven sections delimited by `2 byte` header string, usually containing a 2 character string.

The first three sections has a fixed byte length, this was important because at this point I had no idea
what I was doing and having a fixed length reduced the complexity.
So I decided to read the first `765 byte` section, but I also knew I had to keep reading
the other sections afterwards. This means I need to keep a pointer to which byte in the `.d2s` file I'm currently
reading at. Luckily Go has this really nifty package called `bufio`. This package exports a
buffered reader that wraps the `io.Reader` interface. A buffered reader basically means we're able to read
parts of a file and then continue from that point afterwards, since the `bufio.Reader` keeps a pointer to the
last byte read it's perfect for us.

```go
// Read the character binary from disk. io.File implements io.Reader.
file, err := os.Open("/Users/stekon/go/src/github.com/nokka/d2s-article/nokkasorc")
if err != nil {
    log.Fatal(err)
}

// Create a buffered reader using the file content.
bfr := bufio.NewReader(file)

// Create a byte slice of exactly 765 bytes that can hold the header data.
buf := make([]byte, 765)

// Read the full length of buf length into the buf byte slice.
_, err = io.ReadFull(bfr, buf)
if err != nil {
    log.Fatal(err)
}

fmt.Println(buf)
```

When we print the content of the `buf` byte slice as a string, we can see that it contains just a bunch
of binary gibberish, but there's also the text `Woo!` from which we can conclude that the Diablo II 
developers had a sense of humor.

![header binary string](/images/buf_string.png)

This binary string is no good use for us as is, but from the previously mentioned research, I knew the data
available from the header and their byte lengths. This means that we can use a `struct` in Go with the correct
byte lengths to slot the bytes into place. Below is a struct with the first six data points in the header.
The rest is omitted since the header is really long, but you can find the full struct [here](https://github.com/nokka/d2s/blob/master/character.go#L90).
```go
// Small sample of the first fields of the header.
type Header struct {
	Identifier uint32    // 4 bytes
	Version    uint32    // 4 bytes
	FileSize   uint32    // 4 bytes
	CheckSum   uint32    // 4 bytes
	ActiveArms uint32    // 4 bytes
	Name       [16]byte  // 16 bytes
	Omitted    [729]byte // 729 omitted bytes
}
```

Now that we have a struct to read the buffer into, we can use the `binary.Read` func
from the Go standard library. It takes three arguments.

- The first one is an `io.Reader` that we pass our `buf` to
- The second argument is the [byte order](https://en.wikipedia.org/wiki/Endianness),
which determines the order the bytes are sequenced in, also called Endianness.
- The last argument is the struct we want to read the data into, and this our header struct.

```go
// Create a new instance of the header struct.
headerData := Header{}

// Read the content of the buffer into the header data struct.
err = binary.Read(bytes.NewBuffer(buf), binary.LittleEndian, &headerData)
if err != nil {
    log.Fatal(err)
}

fmt.Println(headerData.Identifier)
fmt.Println(headerData.CheckSum)
fmt.Println(strings.Trim(string(headerData.Name[:]), "\x00"))
```

Ok so now that we have our header data, lets confirm that we have some useful stuff in there,
instead of this binary gibberish. The name is a bit tricky because it's a `[16]byte` to fit
all potential names allowed in game. Our character name is a bit shorter, which means there's a few
empty bytes at the end.

```go
[78 111 107 107 97 83 111 114 99 0 0 0 0 0 0 0]
```

To get rid of these we can simply trim all the trailing empty characters `\x00`. 
Running the program right now will output the identifier, the checksum and the name.

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

```go
for {
    // Read the 9 bit id.
    id, err := br.ReadBits(9)
    if err != nil {
        return err
    }

    // If all 9 bits are set, we've hit the end of the attributes section
    //  at 0x1ff and exit the loop.
    if id == 0x1ff {
        break
    }

    // The attribute bit map is a key value store of bit lengths per
	// attribute ID.
    length, ok := attributeBitMap[id]
    if !ok {
        return fmt.Errorf("unknown attribute id: %d", id)
    }

    // The attribute value.
    attr, err := br.ReadBits(length)
    if err != nil {
        return err
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
	log.Fatal(err)
}

skillHeaderData := skillData{}
err := binary.Read(bytes.NewBuffer(buf), binary.LittleEndian, &skillHeaderData)
if err != nil {
	log.Fatal(err)
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

```go
// Skill represents an available character skill in Diablo II.
type Skill struct {
	ID     int    `json:"id"`
	Points int    `json:"points"`
	Name   string `json:"name"`
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

// Make sure the correct header value is there.
if string(skillHeaderData.Header[:]) != "if" {
	log.Fatal(fmt.Errorf("failed to find skill header"))
}

// Read skill offset based on the character class from the skill offset map.
skillOffset, ok := skillOffsetMap[uint(char.Header.Class)]
if !ok {
	log.Fatal(fmt.Errorf("unknown skill offset for class %d", char.Header.Class))
}

for i, allocatedPoints := range skillHeaderData.List {
	id := (i + skillOffset)
	
	skillName, ok := skillMap[id]
	if !ok {
		return fmt.Errorf("unknown skill id %d", id)
	}

	char.Skills = append(char.Skills, Skill{
		ID:     id,
		Points: int(allocatedPoints),
		Name:   skillName,
	})
}
```

## Item list
This is where it gets seriously complicated and I spent probably around 90% of the time on this part.