# Proposal Process
   (largely stolen from https://github.com/golang/proposal)

   Pilosa's development process is design-driven. 
   Significant changes to the core libraries or tools 
   must be first discussed, and sometimes formally documented, 
   before they can be implemented.

   This document describes the process for proposing,
   documenting, and implementing changes to Pilosa.

   Pilosa's development process is primarily 
   based on the Go language's process.

## Goals
* Make sure that proposals get a proper, fair, timely, recorded evaluation with a clear answer.
* Make past proposals easy to find, to avoid duplicated effort.
* If a design doc is needed, make sure contributors know how to write a good one.

## Definitions
* A proposal is a suggestion filed as a GitHub issue, identified by having the Proposal label.
* A design doc is the expanded form of a proposal, written when the proposal needs more careful explanation and consideration.

## Scope
The proposal process should be used for 
any notable change or addition to Pilosa or official tooling. 
Since proposals begin with the filing of an issue,
even small changes can go through the process if appropriate.
Deciding what is appropriate is a matter of judgement we will
refine through experience. If in doubt, file a proposal.

## Compatibility
By the time Pilosa hits 1.0, we'll have to decide what sort of backwards
compatibility promises we want to make with our APIs.
    

## Process
1. Create an issue (in the appropriate repository of the pilosa organization) describing the proposal.
2. Have an initial discussion:
   * The goal is to (1) accept, (2) decline, or (3) ask for a design doc.
   * The discussion should be resolved in a timely manner.
   * If the author wants to write a design doc, then they can write one.
   * A lack of agreement means the author should write a design doc.
   * If there is disagreement about whether there is agreement, @travis is the arbiter.
3. It's always fine to label a suggestion issue with the Proposal to opt in to this process.
4. It's always fine not to label an issue. (it can be added later if necessary)
5. If a proposal leads to a design doc:
   * The design doc should be checked in to the proposal repository as
     `design/NNNN-shortname.md`, where `NNNN` is the GitHub issue number and
     shortname is a short name (a few dash-separated words at most).
   * The design doc should follow the template(TODO).
   * It is expected that the design doc may go through multiple checked-in revisions.
   * New design doc authors may be paired with a design doc "shepherd" to help work on the doc.
   * For design documents should be wrapped around the 80 column mark. Each
     sentence should start on a new line so that comments can be made
     accurately and the diff kept shorter. Review comments should be
     restricted to grammar, spelling, or procedural errors related to the
     preparation of the proposal itself. All other comments should be
     addressed to the related GitHub issue.
6. Once comments and revisions on the design doc wind down, there is a final
   discussion about the proposal.
   * The goal of the discussion is to reach agreement on (1) accept or (2) decline.
   * The discussion should be resolved in a timely manner.
   * If clear agreement cannot be reached, the arbiter reviews the discussion and makes a decision.
7. The author (and/or other contributors) do the work.
