<div class="date-published">21 Jul 2025</div>

# Secure Communication with Kyber and Dilithium in Rust

## What Is Post-Quantum Cryptography?

Quantum computers are no longer just science fiction—they’re getting real. And with them comes a big challenge: they could break many of the cryptographic systems we rely on today to keep our data safe.

Most current encryption methods, like RSA or Elliptic Curve Cryptography, depend on math problems that are tough for regular (classical) computers to solve. But quantum computers could solve those same problems much faster using special algorithms like Shor’s algorithm. That means the encrypted data we think is secure today might not be safe in the future.

That’s where post-quantum cryptography (PQC) comes in. It’s a new kind of cryptography designed to resist attacks even from quantum computers. Instead of relying on things like factoring or discrete logarithms, PQC algorithms use different mathematical problems, like lattice-based constructions, that are believed to be hard for both classical and quantum computers to crack.

Organizations like NIST (the National Institute of Standards and Technology) have been working on standardizing these new algorithms. Two of the most promising ones are Kyber (used for exchanging keys securely) and Dilithium (used for digital signatures). These are the tools I'll be using in my demo.

It’s also worth noting that NIST has selected an additional algorithm called __Hamming Quasi-Cyclic__ (__HQC__) as a backup to __Module-Lattice based Key Encapsulation Mechanism__ (__ML-KEM__) a.k.a __Kyber__, the primary choice for general-purpose encryption. HQC is based on different mathematical foundations, giving us more options if future research reveals weaknesses in lattice-based methods.

## Digital Signatures and Key Encapsulation Mechanisms (KEMs)
Before I jump into my demo, it’s good to understand two key building blocks: digital signatures and key encapsulation mechanisms (KEMs).

A digital signature lets someone prove that a message (or data) really came from them and hasn’t been tampered with. Think of it like signing a document—except it's done with math. In my demo, the server uses the Dilithium signature scheme to sign its public key, so the client can verify it's talking to the right party and not an imposter.

A key encapsulation mechanism (KEM) is a way for two parties to securely agree on a shared secret over an insecure network. One side publishes a public key, and the other uses it to generate a shared key and send it back in a “capsule” (ciphertext). The owner of the private key can open the capsule and recover the same shared key. In my case, I use Kyber, a lattice-based KEM, to establish this shared secret.

Now, you might be wondering: how is KEM different from regular public-key encryption?

The main difference is that public-key encryption is typically used to encrypt arbitrary messages, while KEM is specifically designed to securely exchange secret keys. It’s a streamlined, more efficient approach for key exchange—especially useful when you want to switch to fast symmetric encryption (like AES) afterward, which is exactly what I do.

##  Communication Flow Using Kyber + Dilithium

To show how post-quantum cryptography can be used in a real-world setting, I’ve created a simple Rust demo that simulates a secure communication session between a client and a server.

For the sake of the example, I pre-generated a Dilithium key pair and stored it in the project directory:

- `keys/dil_pk.txt`: the public key (accessible to anyone)
- `keys/dil_sk.txt`: the secret key (only accessible by the server)

Let’s walk through the communication process.

__Roles:__

- Client 
- Server

---

### Step 1
Client initiates communication.
 
- Client sends communication request (e.g. opens a TCP connection to the Server)
- This is just the initial network setup — no cryptography yet.


### Step 2 

Server sends its Kyber public key + Dilithium signature. Server 

- already has a pair of digital signature (dilithium) keys: `(pk_dil, sk_dil)`
- generates a pair of Kyber keys: `(pk_kyber, sk_kyber)`
- signs its Kyber public key using its Dilithium secret key: `signature = sign(pk_kyber, sk_dil)`
- sends the following payload to the client 

```json
{
    "pk_kyber": "...",
    "signature": "..."
}
```

### Step 3
Client verifies the signature. Client
 
- already has Server's Dilithium public key (`pk_dil`) through some trusted method (e.g., certificate, config, or manual distribution)
- verifies: `verify(pk_kyber, signature, pk_dil)`
- terminates the connection (because the client can’t trust that the key is really from the server) if the client is unable to verify the signature, otherwise the client trusts the server.
- generates and secret key and encapsulates it using Server's Kyber public key: `(ciphertext, shared_secret) = encapsulate(pk_kyber, rng)`
- sends the `ciphertext` to the Server.

### Step 4 
Server decpasulates the ciphertext. Server receives the ciphertext and uses its Kyber secret key to recover the shared secret: `shared_secret = decapsulate(ciphertext, sk_kyber)`

Now both Client and Server share the same secret.

### Step 5
Secure symmetric communication begins. 

---

With the shared secret established, Server and Client switch to symmetric encryption (like AES-GCM) for fast, secure communication.

## Demo: Secure Communication in Rust Using Kyber + Dilithium + AES-GCM
The full example is available in [my GitHub repository](https://github.com/fade2black/kyber-dilithium-demo), but here’s the core part of the demo (the `main.rs` file) which ties everything together.

This simple Rust program demonstrates:

- how a client and server establish a shared secret using Kyber
- how the server proves its identity using Dilithium signatures
- and how both parties securely communicate using AES-GCM symmetric encryption

```rust
use kyber_playground::Server;
use kyber_playground::Client;

fn main() {   
    let mut client = Client::new(); 
    let client_messages = ["Hi!", "What's up!", "Me also good!"];
    let mut server =  Server::new();
    let server_messages = ["Hi!", "All good! You?", "Glad to hear from you!"];
    
    // Client initiates a communication.
    // Server accepts the connection and generates a (kyber public key, signature of the public key) pair. 
    let (pk, sig) = server.generate_pk_and_sig();

    // Server sends the public key and signature to the client over a network.
    // Client recieves the payload, verifies the public key, generates
    // a secret key, encapsulates it, and sends it back to the server.
    let ciphertext = client.accept_pk_and_sig(pk, sig);
    // Server recieves the ciphertext and decapsulates it.
    server.accept_ciphertext(ciphertext);
    // At this stage both client and server have a shared secert key
    // and now can securely communicate using one of secert key algorithms.
    // e.g. AES-GCM
    // Server and client start to communicate.
    for i in 0..3 {
        let plaintext = client_messages[i].as_bytes();
        let (nonce, ciphertext) = client.encrypt_message(plaintext);
        // Client sends the nonce and ciphertext over netwwork to the server 
        let decrypted = server.decrypt_message(&ciphertext, nonce);
        println!("Client: {:?}", String::from_utf8(decrypted).unwrap());

        let plaintext = server_messages[i].as_bytes();
        let (nonce, ciphertext) = server.encrypt_message(plaintext);
         // Server replies with the nonce and ciphertext over netwwork to the client
        let decrypted = client.decrypt_message(&ciphertext, nonce);
        println!("Server: {:?}", String::from_utf8(decrypted).unwrap());
    }                
}
```

This example demonstrates just the high-level flow. If you’re curious about the implementation details—like how keys are loaded from files, how Kyber and Dilithium are integrated, or how AES-GCM is used under the hood—feel free to explore the [full source code](https://github.com/fade2black/kyber-dilithium-demo) on GitHub.

In the next part, I’m planning to expand the demo into a more realistic setup, with separate client and server processes communicating over a network using the TCP protocol. This version will include key exchange and secure message transmission using symmetric encryption (AES-GCM) in a real network environment.