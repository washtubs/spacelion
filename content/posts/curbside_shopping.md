---
title: Curbside Shopping
description: COVID-19 stupid programming project idea
date: 03 May 2020
---

## Purpose

Allow small businesses with no IT infrastructure to rapidly implement a curbside shopping experience for customers,
permitting customers to browse the store
by looking at annotated pictures taken by employees,
and requesting those items without leaving their cars.

## Goals

1. Allow small businesses to create a high fidelity, attractive virtual shopping experience for curbside customers
2. Maximize number of employees who can work from home
3. Rely exclusively on free and open source technology, and even support self-hosting, if businesses choose

## How does it work?

The point of the software is to give users an experience as close as possible to browsing in the store
without physically being there,
while also balancing the workload involved in a business maintaining such a virtual presence.

Businesses need to regularly take pictures of the contents of their shelves and publish them online so users see what is in stock and what isn't.

Employees "groom" the pictures adding metadata as needed so it's as usable as possible

Customers place orders by browsing these pictures and simply tapping what they want.
It's sort of like a shopping cart but there isn't necessarily a price for each item, you just point to what you want.
Customers can also ask for a quantity.

A shopper does a best effort to fulfill the order,
(the app gives them a sequence of pictures of what people pointed to),
and presents the customer with the final tally,
at which point the customer can ask to take some things off.

Once the order is placed a runner can take the items out to the customer's vehicle.

## Quality

IMO picture quality is what will define the quality of the entire experience,
therefore the platform itslef, being associated the that quality, should enforce them externally.

Therefore it is extremely important that customers get the official app, not some self-hosted version.

## Security considerations

Security is quite important because our target users are small businisses which don't necessarily have
the resources like amazon to roll off bad customers by just letting the occasional bad apple have their way.

### Security: **Customer** originating attack

The main attack I'm considering is an *amplification* attack
in which a bad actor causes employees to fill a cart without coming to actually pay for the items,
forcing the employees to return the items to the shelves and restock.

There are different ways to handle this. Let's first consider some bad ideas.

1. **Captchas**: The attacker can easily be manned. It does not have to be an automated attack to be damaging as with most "amplification" attacks.
2. **Enrolling cusomers as users, and using real IDs**: Just too much. Nobody wants to enroll to shop.

Basically the business needs to know, or at least be reasonably sure that a user is legit before employees are required to do any actual real work.

The bottom line is employee authentication is easy (because they enroll), but customer authentication is difficult because they don't.

All authentication methods require the user to "park" or physically enter the lot.

**Zero auth**

Just allow people to place orders from their homes. Do a best effort with captchas etc. This is the gold standard for availability.

Another possibility here is creating skin in the game by requiring customers to input their credit card info up front and requiring a minimum charge of $5
before someone starts working on their order.

**Relaxed auth**

**Parking spots with QR codes**: A shared authentication token can be embedded in a QR code, allowing customers to simply scan the QR code to place an order.

Disadvantage is the bad actor just needs to park to get the authentication code, but if the bad actor is far away and has no connections, it might subdue them.
Advantage is it requires less employee action. A rotation policy can be implemented where you change the authentication code once a month, week, or day.

**Try-hard auth**

**Car make / model / color challenge response**: Prevent users from placing orders until they have declared their car, which can be verified by an employee

**License plate challenge response**: Could have a security camera that reads license plates as they come in

### Security: **Tenant** originating attack

Anyone can be a tenant, so it's very important that tenants have an isolated view of the database.
Each table should have a tenant ID and each tenant should be required to only see records with that ID.

## Maybe we do actually want customer enrollments

If users enroll, they can add their car if it's needed for some places, they can also specify delivery preferences
such as "leave it on the hood", "place it in the trunk etc"

I guess the point is that the "unenrolled" UX should be a good one.

## Pricing and annotations optional

The MVP version of this project is to expose frames to users and allow them to point and click on what they want.
So businesses can sort of do a minimum amount of work, and just take pictures of their shelves and that might be "good enough".
Obviously it's ideal to have product info (especially for ADA),
so this is a feature we should work on immediately once the MVP is released.

Also if the pictures are a good enough quality users can just go off the price tags in the pictures

## Proof of concepts

It must be possible for someone to rapidly take scan QR codes, with sufficient visibility constraints.
The PWA will need to be able to take video, identify QR codes in the view, and instruct the user to move closer, further, or pan to make sure the quality is good.

Another application, possibly not a PWA will need to be able to annotate objects in the frames,
possibly supporting a UPC lookup, from a user typing in the name of what they see,
and autofill the price based on a price sheet.

## Application structure

### Objects

**Note on DB impl:** Whatever the database is it should support multi-tenancy, where a tenant equals a business.
We support self-hosting but we should also have our own database both for businesses who don't want to self host,
and to support a "test-drive" or an ephemeral tenant.

##### Object: Frame

A frame is the representation for a persistent location in a store where a picture needs to be taken, sanctioned off by a plainly visible QR code,
that the business prints and displays to position where the frame should be.

It basically has two states.
When a frame is **bare** it has no version and therefore no picture has been taken yet, so the `currentFrameVersionId` is -1
But the QR code can be printed, as there is always a `code`

    Frame {
        // Internal database identifier
        frameId: int
        currentFrameVersionId: int
        // String encoded into the QR code
        code: string
    }

##### Object: FrameVersion

A frame version is essentially a picture of a frame along with annotation metadata, providing an up to date view of what's in that frame

    // When you take a new picture of a frame a FrameVersion is created, including the previous url and a copy of the annotations
    FrameVersion {
        frameId: int
        frameVersionId: int
        // URL for a picture file, which is made available via CDN
        // The picture itself will have the QR code in it.
        // A new pictureUrl will be assigned each time it's updated
        pictureUrl: string
        annotations: []FrameAnnotation
    }

##### Object: FrameAnnotation

Allows metadata to be associated with a frame, including metadata that is tied to a particular bounding box inside the frame

    FrameAnnotation {
        // OPTIONAL: describes the bounding box of the annotation
        // if not given it applies to the whole frame
        boundingBox: string
        annotations: map[string]string
    }


##### Object: SuperFrame

A wider picture that must include multiple frames. it's used to stitch together multiple frames and can actually
be used to map out a complete adjacency graph of an entire aisle,
as long as the super frames are connected.

They allow the user to zoom in and out, and "browse" an aisle as if they were in the store.

They also allow for routing the shopper on a "shortest path" through the store.

    SuperFrame {
        // The set of frames that are always enclosed by this super frame
        frameIdSet: []int
        pictureUrl: string
    }


##### Object: NeedsWork

    // Encodes information about a FrameVersion that indicates that it needs work
    AnnotationNeedsWork {
        generic
        bad-selection-box
        pricing-update
        label-update
    }

    FrameNeedsWork {
        generic
        price-tag-not-visible
        item-label-obscured
    }

### Roles

##### Role: Customer

Places order.
Amends order.
Confirms order.

##### Role: Shopper

A shopper is responsible for fulfilling orders,
in addition they should update any frames as they go when things go out of stock for example.

##### Role: Scanner

Able to update frames by taking a new picture.

##### Role: Annotater

Adds annotation information to a frame after it's been uploaded. This work can be done remotely,
and therefore shouldn't need to be done by the scanner

##### Role: QA

Acts as a shopper, trying to buy items taking note of things that are out of place.
Able to flag frames as needing attention for various reasons, such as price-tag-not-visible, item-label-obscured.
Able to do other grooming tasks such as delete unnecessary super frames

## Views

##### View: Scan Frame

*Role*: Scanner

Allows selecting a frame for various forms by physically scanning with the camera
Once the camera focuses on a QR code, that's all that's needed.
Just returns a frame ID.

##### View: Update Frames

*Role*: Scanner

This view simply takes an up-to-date picture of a frame and stores a new FrameVersion immediately
However this is probably the most complicated view
Each picture needs to conform to certain quality constraints, so there needs to be feedback to the user

##### View: Browse Frames

*Role*: Customer

### User Stories

##### User Story: Enroll an org

An owner can enroll their org, with our existing infrastructure and get their own tenant ID
or if it's their own infra, they can use the default ID

##### User Story: Enroll an employee

An employee can enroll with an email or username + password.

Information the employee needs to fill out:
1. Nick name

Owner can supply each employee with roles

##### User Story: Create frame(s)


##### User Story: Take a superframe

##### User Story: Updating a frame

When a shopper sees something has gone out of stock, it's important to update the frame so customers can see.

1. The shopper (S) opens the PWA, which immediately starts taking video
2. S positions the video until the QR code is in the correct position, and a photo is automatically taken
3. S is asked to check the if the old annotations and bounding boxes fit what is shown in the new photo, so as to inherit the previous metadata as much as possible
4. They can confirm or flag it as needs work, saying specifically what's wrong or just letting the annotater figure it out with a generic flag
5. If it is flagged at this stage, an annotater will need to come in and update it before the new version rolls
6. The version is rolled, and users browsing that frame will be notified that it was updated.

##### User Story: Customer places a simple order

##### User Story: Customer places an order but something is out of stock

##### User Story: Customer amends an order after the shopper has started

##### User Story: Customer removes things from order before checkout

## Will I fuck this up

I have no retail experience so on a scale of 1 to 10, yes

