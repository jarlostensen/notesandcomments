The title of this post started out as  “A Swift implementation of binary arithmetic in GF(2)^n” but I decided against it because, even though it’s more accurate, it defeats the purpose of the post which is to (hopefully) explain the principles in-code and in human-readable form...

For your perusal; here is some background, references, and more formal details;

http://xcore.github.io/doc_tips_and_tricks/crc.html

https://sites.google.com/site/ctxtree/crc/crc-binarydivision

https://en.wikipedia.org/wiki/Modular_arithmetic

http://jameskbeard.com/Temple/Data/Binary_Polynomial_Division.pdf

Now, to divide two binary numbers, modulo 2, you use the same technique as “long division” but I’ve not done long division by hand since a very long time and quite frankly can’t remember much of it so I am not going to use that excuse and instead present you directly with an algorithm, firstly in pseudo-code, then in Swift;

divideBinaryModulo2(N, D) where N>D
 let divisor = msb of D aligned with the msb of N by 
               shifting it up k bits
 let dividend = N
 let quotient = 0
 let remainder = 0
 for i = k-1..0 {
   if high bit of dividend is set
     set i’th bit of quotient to 1
     dividend = dividend XOR divisor

   shift dividend up 1 bit
 }
 remainder = dividend shifted back down k bits

The result is the quotient and remainder such that

quotient * D + remainder = N

Addition and subtraction is the same, modulo 2, and is implemented with the XOR operation so another way of writing this is

quotient * D XOR remainder = N

Where multiplication of A and B, modulo 2, is done by summing A multiplied by each term of B, i.e;

multiplyBinaryModulo2(A,B)
 result = 0 
 for each set bit in B {   
   let i = bit position
   result = result XOR (A shifted up by i)
}

Note; For the multiplication to be correct we also need to ensure that the result doesn't go beyond 32 bits and the way to do this is to do the multiplication modulo a primitive polynomial of order 32 (i.e. one that uses 33 bits.)

Imagine we wanted to work within only 8- of our 32 -bits. Since addition is an XOR operation, assuming the terms are of order less than 8, it can never bring a result to overflow. Multiplication, however, could easily do that and the question then is; given the result of a polynomial multiply which overflows our 8 bits, what do we do...?

One approach might be to just AND it by 0x7f (0b1111111) to mask out the upper bits. This certainly works but it's not going to give us the right results because, firstly, it's a logical operation (not an arithmetic one that's either a + a - or a *, which is required for this to remain inside the confines of a "ring" - which is a bit like a mathematical "group" a little more structure, like the concept of an "inverse"), secondly it's not guaranteed to give a non-0 result and 0 isn't a polynomial (it's just a constant) and we don't want it to show up in our results. Fixing the overflow also needs to give us a result which can be any of the possible polynomials we can fit in the 8 bits; i.e. it shouldn't rule any out and thereby reduce our set of polynomials.

To the rescue; prime numbers. Or, to be more specific, prime polynomials. Something is prime if it's not a product of other parts so a prime polynomial, or a primitive-, and irreducible  -polynomial can't be written out as the product of other, lower order, polynomials. Another important property of prime polynomials (or numbers) is that they guarantee that a "multiplicative inverse" exist in a collection where we define multiplication as including a modulo on the result; i.e. a*b = a*b mod p, where p is the prime (number or polynomial).

To be more specific; since a prime number can't be written as the product of two or more, smaller, numbers the product a*b can never be a multiple of that (or any other) prime number. This means that taking the modulo will never result in 0, since you only get 0 if the remainder is 0, i.e. if a*b is a multiple of p...The same logic applies to p being a prime (irreducible, primitive) polynomial.

So; to make our multiplications correct and mathematically consistent for polynomials, and to fit inside our example of 8 bits we need to follow every multiplication with a modulo (i.e. remainder) operation on the result by a prime polynomial. This prime polynomial can be picked relatively arbitrarily and there are tables for these (they can also be found using the same techniques used to find prime numbers. I've tried this with the Sieve of Eratosthenes and I'll publish this addition to the code below as soon as I have time.) So, therefore; the code below is, strictly, not correct for multiplications.

Now for the Swift part;

I’ve created a small class GfPolynomial32 which contains the code for performing modular arithmetic on bit strings up to 32 bits in length. The reason for the “Polynomial” part in the name, btw, is because another way of representing these bit strings is as polynomials of a parameter x which have either a “1” or a “0” coefficient. I.e., given the bit string 10110 we can interpret this as;

 1*x^4+0*x^3+1*x^2+1*x+0

I’ve added some extension methods to create string representations of the bits either as conventional 1’s and 0’s or in the polynomial representation.

Here, without further ado, is the code (just remember that it has not been written for speed and performance, but rather for understandability)

//

//  Gf2Polynomial32.swift

//

//  Created by Jarl Ostensen on 07/01/02015.

//  Copyright (c) 2015 SonarJetLens. All rights reserved.

//

import Foundation

// Gf2 representation of a polynomial of maximum order 31

public class Gf2Polynomial32 {

    private var _value:UInt32;

    private let _valueMsbPos:UInt32 = 0;

    // return the position of the highest set bit in value

    private func msbPosOf(Value:UInt32) -> UInt32 {

        var pos:UInt32 = 0;

        var c = Value;

        while(c != 0) {

            c >>= 1;

            ++pos;

        }

        return pos - 1;

    }

    public init(val:UInt32) {

        _value = val;

                  _valueMsbPos = msbPosOf(_value);

    }

    // order of the polynomial (equals position of msb)

    public var order:UInt32 {

        get {

            return _valueMsbPos;

        }

    }

    // raw value

    public var value:UInt32 {

        get {

            return _value;

        }

    }

}

// ====================================== various operators on Gf2Polynomial32's;

public func == (left:Gf2Polynomial32, right:Gf2Polynomial32) -> Bool {

    return left.value == right.value;

}

public func != (left:Gf2Polynomial32, right:Gf2Polynomial32) -> Bool {

    return left.value != right.value;

}

public func + (left:Gf2Polynomial32, right:Gf2Polynomial32) -> Gf2Polynomial32 {

    return Gf2Polynomial32(val:(left.value ^ right.value));

}

public func - (left:Gf2Polynomial32, right:Gf2Polynomial32) -> Gf2Polynomial32 {

    // NOTE: same as for +

    return Gf2Polynomial32(val:(left.value ^ right.value));

}

// multiplication

public func * (left:Gf2Polynomial32, right:Gf2Polynomial32) -> Gf2Polynomial32 {

    var b:UInt32 = right.value;

    let a:UInt32 = left.value;

    var result:UInt32 = 0;

    var shift:UInt32 = 0;

    while(b != 0) {

        if ( b&1 == 1 ) {

            result ^= (a << shift);

        }

        ++shift;

        b >>= 1;

    }

    return Gf2Polynomial32(val:result);

}

// division (returns a pair; quotient and remainder)

public func / (left:Gf2Polynomial32, right:Gf2Polynomial32) -> (Gf2Polynomial32,Gf2Polynomial32) {

    let shiftAlign = (left.order - right.order);

    let divisor = right.value << shiftAlign;

    let highBitMask:UInt32 = 1 << left.order;

    var q:UInt32 = 0;

    var dividend = left.value;

    var qShift = shiftAlign+1;

    do {

        if ( (dividend&highBitMask) != 0 ) {

            dividend ^= divisor;

            q ^= (1 << (qShift-1));

        }

        --qShift;

        dividend = (dividend << 1);

    } while( qShift != 0 );

    return (Gf2Polynomial32(val:q),Gf2Polynomial32(val:(dividend>>(shiftAlign+1))));

}

And some helpful extensions;

//

//  Gf2Polynomial32+Extension.swift

//

//  Created by Jarl Ostensen on 07/01/02015.

//  Copyright (c) 2015 SonarJetLens. All rights reserved.

//

import Foundation

extension Gf2Polynomial32 {

    // return a 1's and 0's representation

    func asBinaryString() -> String {

        if ( value == 0 ) {

            return "0";

        }

        else {

            var result = "";

            var c = value;

            while( c != 0 ) {

                if ( (c & 1) != 0 ) {

                    result = "1" + result;

                }

                else {

                    result = "0" + result;

                }

                c >>= 1;

            }

            return "0b" + result;

        }

    }

    // return a polynomial representation

    func asPolynomial() -> String {

        if ( value==0 ) {

            return "0";

        }

        else {

            var result = "";

            var c = value;

            var pos = 0;

            while( c != 0 ) {

                if ( (c & 1) != 0 ) {

                    if ( pos>0 ) {

                        result = "x^\(pos)" + (result.utf16Count>0 ? " + \(result)" : "");

                    }

                    else {

                        result = "1";

                    }

                }

                c >>= 1;

                ++pos;

            }

            return result;

        }

    }

}

And finally; here is some example output from using this class;

N = 0b1100110111; x^9 + x^8 + x^5 + x^4 + x^2 + x^1 + 1

D = 0b10011; x^4 + x^1 + 1

q = 0b110110; x^5 + x^4 + x^2 + x^1

r = 0b1101; x^3 + x^2 + 1

q*D     = x^9 + x^8 + x^5 + x^4 + x^3 + x^1

q*D + r = x^9 + x^8 + x^5 + x^4 + x^2 + x^1 + 1
