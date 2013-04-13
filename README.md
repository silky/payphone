##Payphone Project

These are some notes from my project to install a working payphone in my apartment and configure it to make and receive calls through an Asterisk PBX.

[![](http://i.imgur.com/N9jgE3Wl.jpg)](http://imgur.com/N9jgE3W) [![](http://i.imgur.com/pq4ABcfl.jpg)](http://imgur.com/pq4ABcf)

####The Phone

This is a Western Electric/AT&T 1D2 single-slot "dumb" payphone, popular in the 1980s.  It came with a T-key but no upper or lower locks or keys.  The handset was pretty gross and there were no upper or lower instruction cards.  Shipping was a bit expensive since the phone weighs nearly 50 pounds.

I purchased a new [handset](http://www.payphone.com/Standard-Handset.html), [mounting backplate](http://www.payphone.com/Mounting-Backplate.html), and [mounting studs](http://www.payphone.com/Brass-Mounting-Stud.html).  I located some old instruction cards on eBay and upper and lower locks with key sets.

The instruction cards install by unscrewing a very tiny allen bolt in the front of the housing to allow the card to slide up and then back down.

####Mounting

To mount the phone to my wall, I drilled 3 holes into the brick and secured the backing plate to the wall with masonry anchors.  The mounting studs hand-screwed into the back of the phone which allowed the phone to easily hang on the backplate, aligning the 12 holes in the back of the phone to the 1/4x20 threaded holes in the backplate.

The phone is mounted at the standard height of 63" from the floor to the top of the housing.

####Connectivity

Back when all payphones were owned by the phone company, a POTS line provisioned for a payphone provided dialtone to the phone.  When coins were inserted, the phone's totalizer sent simultaneous 1700+2200 Hz tones down the line for each 5 cents (nickel = one 66 ms tone, dime = two 66 ms tones with a 66 ms pause in between, quarter = five 33 ms tones with 33 ms pauses).

The phone company's Automated Coin Toll Service (ACTS) would respond to the tones generated by the phone (or your [red box](https://en.wikipedia.org/wiki/Red_box_%28phreaking%29)) and allow the dialed phone number to connect for a certain amount of time.

Newer "smart" (Elcotel-style) payphones are commonly owned by private companies (Customer Owned Coin Telephones - COCOTs) and don't require a specially-provisioned phone line.  The phone has an embedded circuit board that has to be programmed with rate information, allowing the phone itself to determine whether the call is allowed to go through based on the number dialed and the amount of coins inserted.  These phones would have to get reprogrammed every so often by a company that calls into the pay phone and uploads new rate information to it.  (If you're looking to acquire a payphone for personal use, avoid these "smart" phones.  An easy way to tell them apart from the outside is that "dumb" phones have the coin slot on the left side and "smart" phones usually have it on the right.  Inside it's easy to tell; just look for a big modern-looking circuit board.)

Since this phone has always worked with a normal POTS line, making it work now just requires hooking up the red and green wires of an RJ11 cable to the Ring and Tip terminals in the phone.  It rings, dials, and otherwise functions just like a normal analog telephone.

I connected the phone to a Grandstream HT701 SIP ATA and was able to make and receive calls through an Asterisk soft PBX, with dialtone coming from the ATA.  I registered an inbound number through Twilio (appropriately enough, one ending in -2600) and routed it to the Asterisk server.  Outbound SIP is handled through an existing SIP peer.

Since this a payphone, after all, it should require depositing coins to make a call.  Much older phones (like 3-slot rotary phones) required coins to be deposited before hearing a dial tone.  These were mostly phased out by the 1970s and replaced with phones that provided a dial tone first, allowing emergency calls without depositing coins as well as depositing extra coins for long-distance calls.  Using Asterisk, this payphone will be configured to provide a dial tone first and allow free emergency calls, but require 25 cents to call any other number.

####Sending Coin Tones to Asterisk

Since the ATA only establishes audio between the phone and Asterisk after a recognizable pattern of digits has been dialed (and sent to Asterisk all at once in one INVITE request), Asterisk would not be able to respond to coin tones generated before dialing.

Using the ATA's "Offhook Auto-Dial" feature, I configured it to automatically (silently) dial `0` as soon as the handset was picked up.  An AGI script on the Asterisk server would then take over, generating its own dialtone and responding to any tones sent by the phone.

*Relevant Asterisk `sip.conf` configuration for the ATA:*

	[2600]
	canreinvite=no
	callerid=<...>
	context=payphone-totalizer
	dtmfmode=inband
	host=dynamic
	nat=yes
	progressinband=no
	qualify=3000
	secret=...
	type=friend
	disallow=all
	allow=ulaw
	allow=alaw

*Relevant Asterisk `extensions.conf` configuration to take over the call as soon as `0` is dialed by the ATA:*

	[payphone-totalizer]
	exten => 0,1,Answer
	exten => 0,2,AGI(payphone.agi)
	exten => 0,3,Hangup

####Recognizing Coin Tones

Now that Asterisk is receiving the 1700+2200 Hz tones generated when coins are inserted, some code is needed to actually recognize them.  Using Asterisk EAGI would allow a program to read the raw audio stream and analyze it for the proper tone frequencies using the [Goertzel algorithm](https://en.wikipedia.org/wiki/Goertzel_algorithm), but doing so would be pretty complicated.

Since Asterisk is already doing in-band dual-tone multi-frequency (DTMF) detection for numerical digits, modifying it to recognize the coin tones is much less complicated.  [This patch to Asterisk's DSP module](asterisk-dsp_recognize_coins.patch) recognizes the 1700+2200 Hz tone and turns it into a `$` digit, allowing any AGI or other code handling numeric digits to easily recognize coin tones.

*A small AGI script to recognize and accumulate inserted coin amounts, printing it to the Asterisk console:*

	#!/usr/bin/perl
	
	use Asterisk::AGI;
	use strict;
	
	my $inserted = 0;
	my $dialed = "";
	my $AGI = new Asterisk::AGI;

	# generate dialtone while we wait for coins
	$AGI->exec("Playtones", "dial");
	
	while (1) {
	    my $digit = $AGI->wait_for_digit(1000);
	    next if ($digit <= 0);
	
	    if ($digit == ord('$')) {
	        $inserted += 5;
	        $AGI->verbose("got 5-cent tone (now " . $inserted . " cents)", 5);
	    }
	    else {
	        $dialed .= chr($digit);
	        $AGI->verbose("dialed " . chr($digit) . " (now " . $dialed . ")", 5);
	    }
	}

Now that the amount of inserted coins can be recognized, along with any digits dialed, it's possible to make a full script that can refuse to connect calls and play an error message until a certain amount of money is inserted.  Toll-free numbers, 911, etc. can be connected before any coins are collected.

My routing script is [under development here](payphone.agi).

####TODO

- Make the coin hopper queue up coins when inserted rather than immediately dropping them into the coin box, to allow for refunding.  This [requires sending high voltage](http://oldphoneguy.net/images/MPPwk.pdf) ([2](http://atcaonline.com/controller.html)) to the coin relay, and would have to be done out-of-band.

- Sometimes Asterisk only recognizes 3 of the 5 tones for a quarter, probably due to the VoIP stream having to go over the Internet.  Since the only way 3 tones can get played in such a short duration is from a quarter, the script should recognize 3 or more tones in a short duration as a quarter and just ignore the other 2 if they are heard.
