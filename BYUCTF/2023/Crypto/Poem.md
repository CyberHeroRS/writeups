Author: lambdanuu

# Prompt

>epcndkohlxfgvenkzcllkoclivdckskvpkddcyoceipkvrcslkdhycbcscwcsc

>Spaces and punctuation were removed :)

# Solution
<p> I've spent way too much time on this overthinking about _poem code_ but this ended up being much easier than expected. </p>
<p> Using <a href="https://www.dcode.fr/cipher-identifier">this</a> site i was able to identify some of the possible cyphers, including </p>
<ul>
    <li>Phillips Cipher</li>
    <li>Keyboard Change Cipher</li>
    <li>Base62 Encoding</li>
</ul>
<p> The most obvious one to try seemed to be the keyboard change cipher. Decoding the result gives ABC...Z->qwerty and the following string: </p>

>thefragisbyuctfamessagesocrealacharrengetohackelsarinewelevele

<p> Or expanded: </p>

>the frag is byuctf a message so creal a charrenge to hackels a rine we levele

<p> After noticing that 'frag' should be 'flag' and 'creal' makes more sense as 'clear' I tried exchanging r and l and got the following string: </p>

>the flag is byuctf a message so clear a challenge to hackers a line we revere

<p> And we submit the flag </p>

>byuctf{a message so clear a challenge to hackers a line we revere}
