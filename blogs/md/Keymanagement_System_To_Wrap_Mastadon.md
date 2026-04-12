### A Key Management System To Wrap Mastadon

#### Introduction

This project was created for my Advanced Computer Networks module. I wrote this system in zig that wraps the social media service Mastadon. The purpose of the project is to allow users to communicate securely through Mastadon with features such as private groups and direct messages.

#### Architecture

The first problem out of the gate was finding an instance of Mastadon that has a larger character limit than 500 chars. I looked for a few hours but couldnt find any public ones so I hosted my own Mastadon instance on my server. This allowed me to changed the character limit to 1,000,000 and create 3 accounts: Alice, Bob and Carol.

Now that I had my transport system working I used the Mastadon API to authenticated users, get users, send messages and recieve messages. To use the program the user runs the application, gives the client thier username and authenticates. Then they are brought to a terminal ui were they can type given commands:
```
msgs                           -> updates messages on client
dm <user> <msg>                -> direct messages another user by their username
gm <group> <msg>               -> group messages a group they are a member of by the name of the group
create <group>                 -> creates a group
inv <user> <group>             -> invite a user to a group they are a member of by their username
rm <user> <group> <group name> -> removes a user from a group by creating a new group with a new name and inviting all users that new group
clear                          -> clears the logs
quit                           -> quits the program and saves
```
This allows the use to communicate with users privately directly or in a group they can create.

#### Communication Overview

As an example of how the user communcicates if Alice wants to direct message Bob:

##### **Alice**:
- Key Generation: Alice generates a long-term key pair.
- Public Key Publication: She publishes her long-term public key to her profile.
- Ephemeral Key Generation: She generates an ephemeral key pair and sends the public ephemeral key in her message.
- AES Key Generation:
  - She computes the AES key using her ephemeral private key and Bob's long-term public key.
  -  She hashes the result and uses the hash as the key for AES encryption.
- Nonce and Preparation: She performs necessary actions (like generating a nonce) for encryption.
- Secure Messaging: Alice can now send messages securely to Bob using the AES key, she encodes with base32 to get printable characters that are url safe and mastadon hashtag safe.

##### **Bob**:

- Key Generation: Bob generates a long-term key pair.
- Public Key Publication: He publishes his long-term public key to his profile.
- Receiving Message: He receives a message from Alice containing her public ephemeral key.
- AES Key Computation:
  - Using Alice's public ephemeral key and his private long-term key, he computes the AES key.
  - He hashes this result to obtain the true key and can proceed from there.

Now if Alice wants to make a group with Bob:

##### **Alice**:
- Key Generation:
- AES Key Generation: Alice generates a random AES key.
- Nonce and Preparation: She performs necessary actions (like generating a nonce) for encryption.
- Secure Messaging: Alice can now send messages securely trough their already cryptographiclly secure direct messages to Bob sending him the AES Key for the group. She encodes with base32 to get printable characters that are url safe and mastadon hashtag safe.

##### **Bob**:

- Receiving Message: He receives a message from Alice containing the group AES key and stores it for use.

#### Data Model

In order make the client convient I had to store information between session. I chose to store this in json as there is a very convinet struct to json methods in the zig standard library.

These are the structs used to store the information:

```zig
pub const Group = struct {
    name: []const u8,
    key: crypto.AesKey,
};

pub const Dm = struct {
    name: []const u8,
    key: crypto.AesKey,
};
pub const GroupPair = struct {
    id: crypto.UUID,
    group: Group,
};

pub const DmPair = struct {
    id: crypto.UUID,
    dm: Dm,
};

pub const ClientSave = struct {
    name: []const u8,
    access_token: []const u8,
    long_term_keys: crypto.X25519.KeyPair,
    groups: []const GroupPair = undefined,
    dms: []const DmPair = undefined,
};
```

It generates json in this sort of format:

```json
{
  long-term-keys: {
    public-key: <key>
    private-key: <key>
  },
  groups: [
    group-pair: {
      key: <aes-key>,
      name: <name>,
    },
    ...
  ],
  dms: [
    dm-pair: {
      key: <aes-key>,
      name: <name>,
    },
    ...
  ]
}
```

One of the things I could have done after doing this project is to group dms and groups togher into one system because they are basically the same. The only difference is how they generate the keys. This simplicity is due to not having admins. This was because I only have clients and not one cenetral server to moderate things.

#### Message Protocol Formats

There are 4 different protocol message formats:
```
    dm_message,
    dm_invite,
    group_message,
    group_invite,
```

##### dm_message:

```
Encoded in base32: {
  Nonce: <nonce>,
  Tag: <tag>,
  Encrypted (DM Messaging Key): {
    Label: [DM Message],
    Message: <msg>
  }
}
#private-zocial-<ID>
```

##### dm_invite:

```
Encoded in base32: {
  Ephemeral Key: <key>,
  Nonce: <nonce>,
  Tag: <tag>,
  Encrypted (Receivers User Long Term Public Key): {
    Label: [DM Invite]
    ID: <new ID for communicating>
  }
}
#private-zocial-0
```

##### group_message:

```
Encoded in base32: {
  Label: [Group Message],
  Encrypted (Group Message Key): {
    Message: <Text message>
  }
}
#private-zocial-<ID>
```

##### group_invite:
```
{
  Encrypted (DM Messaging Key): {
    Label: [Group Invite],
    Group Message Key: <key>
  }
}
#private-zocial-<ID>
```

With only these 4 formats we can achieve all the communication we need for this project.

#### Links

Here is the linke to the code on github: [https://github.com/ThomasCreagh/private-zocial](https://github.com/ThomasCreagh/private-zocial)
