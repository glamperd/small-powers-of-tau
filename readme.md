# Small Powers of Tau

**This code has not been auditted. Use at your own risk**

This projects aims to implement a small non-Groth16 powers of tau ceremony. It is advised to only use this implementation, if your ceremony is less than 2^18 or if your computer is powerful.

## Performance

- No optimisations have been added, so the code will most likely be very slow. This includes adding rayon and swapping many pairings checks for multi exponentiations and a single pairing check.
## Usage

Below we show different uses of the API depending on the actor.

### Contributor

In order to contribute to a ceremony, you need to:

- Receive and deserialise the most recent SRS
- Create a random private key
- Use your private key to update the SRS, creating an update proof
- Send the update proof and the updated SRS.

```rust
    // Specify the parameters for the ceremony
   let params = Parameters {
            num_g1_elements_needed: 100,
            num_g2_elements_needed: 2,
    };

    let bytes = // Assuming you received the most recent SRS as a bytes
    
    // Deserialise the bytes received to create the SRS
    // This method will ensure that the points are in the correct group and that none of the points are zero
    let mut srs = Accumulator::deserialise(bytes, params);

    // Create your private key
    let private_key = PrivateKey::rand(rng);

    // Update the SRS creating an update proof 
    let update_proof = srs.update(private_key);

    // Send the SRS and update proof to the appropriate party
    update_proof.serialise();
    srs.serialise();
````

### Protocol Verifier

The job of the protocol verifier is to check whether a contribution was valid during the ceremony. If not, the contribution is thrown away like it never existed along with the update proof.

The workflow is as follows:

- Receive and deserialise the SRS that was sent by a contributor along with the update proof
- Check that the update proof is valid. This includes checking that the SRS received, does indeed build on the SRS that the verifier currently holds.
- Signal to the protocol if the update was valid. At this point, one could delete the old SRS keeping the update proof.

```rust
    // Specify the parameters for the ceremony
   let params = Parameters {
            num_g1_elements_needed: 100,
            num_g2_elements_needed: 2,
    };

    let bytes = // Assuming you received the most recent SRS and the update proof

    let srs_old = // The verifier will always have an old SRS in memory or on disk

    // Deserialise the bytes received to create the SRS
    // This method will ensure that the points are in the correct group and that none of the points are zero
    let mut srs_new = Accumulator::deserialise(bytes, params);
    // Deserialise the update proof
    let update_proof = UpdateProof::deserialise(bytes)

    let valid_update = Accumulator::verify_update(&srs_old, &srs_new, &update_proof);

    // Do something based on whether the update was valid
````

### Ceremony Integrity Verifier

These are the actors who want to either check that their contributions were included in the SRS or that the SRS was indeed updated according to the update proofs.

```rust
    // Specify the parameters for the ceremony
   let params = Parameters {
            num_g1_elements_needed: 100,
            num_g2_elements_needed: 2,
    };

    let bytes = // Assuming you downloaded the starting SRS, the final SRS and the update proofs from some storage location

    let starting_srs = Accumulator::deserialise(bytes, params);
    let final_srs = Accumulator::deserialise(bytes, params);
    let update_proofs = //

    // Verify that these update proofs indeed do correspond to the transition from the starting SRS to the final SRS
    let valid_updates = Ceremony::verify(starting_srs, final_srs, update_proofs);

    // Now lets assume that I have contributed to the ceremony and I want to verify tht my contribution was included.
    // I should have a public key that was included in my update proof
    let public_key = //

    let (valid_updates, position) = Ceremony::verify_and_find_contribution(starting_srs, final_srs, update_proofs, public_key);

    // If position is None/Nil then your contribution was not included. Else the position of your contribution will be returned.
````