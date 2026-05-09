# Common arithmetic operations in bignumber.js

For on-chain amounts, token decimals, `wei`, and similar scenarios, do not first represent the value as a JavaScript `number` and then pass it to BigNumber. A `number` may already have suffered binary floating-point precision loss, and BigNumber can only preserve the value after it has been passed in. In real projects, prefer strings, integer smallest units, or integer types returned by contracts or SDKs when constructing BigNumber values.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>2</title>
  </head>
  <body>
    <div class="main"></div>

    <script src="./bignumber.js"></script>
    <script>

      let n = 126.549876212;

      let x = BigNumber(n);
      console.log(x);
      // Check whether the type is BigNumber.
      console.log("Check type => BigNumber.isBigNumber(n) Result:", BigNumber.isBigNumber(n));
      console.log("Check type => BigNumber.isBigNumber(x) Result:", BigNumber.isBigNumber(x));
      // Get the maximum value.
      console.log("Maximum => BigNumber.max(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.max(12, 2, 3, 34, 55).toNumber());
      console.log("Maximum => BigNumber.maximum(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.maximum(12, 2, 3, 34, 55).toNumber());
      // Get the minimum value.
      console.log("Minimum => BigNumber.min(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.min(12, 2, 3, 34, 55).toNumber());
      console.log("Minimum => BigNumber.minimum(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.minimum(12, 2, 3, 34, 55).toNumber());
      // Sum.
      console.log("Sum => BigNumber.sum(4e9, x, '1').toNumber() Result:", BigNumber.sum(4e9, x, "1").toNumber());
      // Random number below 1 with 2 decimal places.
      console.log("Random below 1 with 2 decimal places => BigNumber.random(2).toNumber() Result:", BigNumber.random(2).toNumber());
      // Absolute value.
      console.log("Absolute value => BigNumber('-1.21').abs().toNumber() Result:", BigNumber("-1.21").abs().toNumber());
      // Compare values.
      // 1    if this BigNumber is greater than n
      // -1   if this BigNumber is less than n
      // 0    if this BigNumber has the same value as n
      // null if this BigNumber or n is NaN
      console.log("Compare => BigNumber(111111).comparedTo(2222) Result:", BigNumber(111111).comparedTo(2222));
      // Round.
      // Three decimal places.
      console.log("Round to three decimal places => BigNumber(12.32162).decimalPlaces(3).toNumber() Result:", BigNumber(12.32162).decimalPlaces(3).toNumber());
      console.log("Round to three decimal places => BigNumber(12.32162).dp(3).toNumber() Result:", BigNumber(12.32162).dp(3).toNumber());

      // Round to an integer.
      console.log("Round to integer => BigNumber(123.756).integerValue().toNumber() Result:", BigNumber(123.756).integerValue().toNumber());

      // Equal to.
      console.log("Equal => BigNumber(123).isEqualTo(123.00000000000000001) Result:", BigNumber(123).isEqualTo(123.00000000000000001));
      console.log("Equal => BigNumber(123).isEqualTo(123.100000000000000001) Result:", BigNumber(123).isEqualTo(123.100000000000000001));
      console.log("Equal => BigNumber(123).eq(123.00000000000000001) Result:", BigNumber(123).eq(123.00000000000000001));
      console.log("Equal => BigNumber(123).eq(123.100000000000000001) Result:", BigNumber(123).eq(123.100000000000000001));

      // Greater than.
      console.log("Greater than => BigNumber(0.3).minus(0.2).isGreaterThan(0.1) Result:", BigNumber(0.3).minus(0.2).isGreaterThan(0.1));
      console.log("Greater than => BigNumber(0.3).minus(0.2).gt(0.1) Result:", BigNumber(0.3).minus(0.2).gt(0.1));

      // Greater than or equal to.
      console.log("Greater than or equal to => BigNumber(0.3).minus(0.2).isGreaterThanOrEqualTo(0.1) Result:", BigNumber(0.3).minus(0.2).isGreaterThanOrEqualTo(0.1));
      console.log("Greater than or equal to => BigNumber(0.3).minus(0.2).gte(0.1) Result:", BigNumber(0.3).minus(0.2).gte(0.1));

      // Less than.
      console.log("Less than => BigNumber(0.3).minus(0.2).isLessThan(0.1) Result:", BigNumber(0.3).minus(0.2).isLessThan(0.1));
      console.log("Less than => BigNumber(0.3).minus(0.2).lt(0.1) Result:", BigNumber(0.3).minus(0.2).lt(0.1));

      // Less than or equal to.
      console.log("Less than or equal to => BigNumber(0.3).minus(0.2).isLessThanOrEqualTo(0.1) Result:", BigNumber(0.3).minus(0.2).isLessThanOrEqualTo(0.1));
      console.log("Less than or equal to => BigNumber(0.3).minus(0.2).lte(0.1) Result:", BigNumber(0.3).minus(0.2).lte(0.1));

      // Addition.
      console.log("Addition => BigNumber(0.2).plus(0.1).toNumber() Result:", BigNumber(0.2).plus(0.1).toNumber());
      // Subtraction.
      console.log("Subtraction => BigNumber(0.3).minus(0.1).toNumber() Result:", BigNumber(0.3).minus(0.1).toNumber());
      // Multiplication.
      console.log("Multiplication => BigNumber(0.3).multipliedBy(0.2).toNumber() Result:", BigNumber(0.3).multipliedBy(0.2).toNumber());
      console.log("Multiplication => BigNumber(0.3).times(0.2).toNumber() Result:", BigNumber(0.3).times(0.2).toNumber());
      // Division.
      console.log("Division => BigNumber(12).dividedBy(7).toNumber() Result:", BigNumber(12).dividedBy(7).toNumber());
      console.log("Division => BigNumber(12).div(7).toNumber() Result:", BigNumber(12).div(7).toNumber());
      // Integer division.
      console.log("Integer division => BigNumber(12).dividedToIntegerBy(7).toNumber() Result:", BigNumber(12).dividedToIntegerBy(7).toNumber());
      console.log("Integer division => BigNumber(12).idiv(7).toNumber() Result:", BigNumber(12).idiv(7).toNumber());

      // Remainder: 10 % 3.
      console.log("Remainder => BigNumber(10).modulo(3).toNumber() Result:", BigNumber(10).modulo(3).toNumber());
      console.log("Remainder => BigNumber(10).mod(3).toNumber() Result:", BigNumber(10).mod(3).toNumber());

      // Negation: -1.3 * -1.
      console.log("Negation -1.3 * -1 => BigNumber(-1.3).negated().toNumber() Result:", BigNumber(-1.3).negated().toNumber());

      // Exponentiation.
      console.log("Exponentiation 2^10 => BigNumber(2).exponentiatedBy(10).toNumber() Result:", BigNumber(2).exponentiatedBy(10).toNumber());
      console.log("Exponentiation 2^10 => BigNumber(2).pow(10).toNumber() Result:", BigNumber(2).pow(10).toNumber());

      // Whether the value is finite.
      console.log("Finite =>  Result:", BigNumber(123).isFinite());
      console.log("Finite =>  Result:", BigNumber(Infinity).isFinite());

      // Whether the value is an integer.
      console.log("Integer =>  Result:", BigNumber(0.3).isInteger());
      console.log("Integer =>  Result:", BigNumber(3).isInteger());

      // Whether the value is NaN.
      console.log("NaN =>  Result:", BigNumber(NaN).isNaN());
      console.log("NaN =>  Result:", BigNumber(Infinity).isNaN());

      // Whether the value is negative.
      console.log("Negative =>  Result:", BigNumber(-0).isNegative());
      console.log("Negative =>  Result:", BigNumber(0).isNegative());

      // Formatting.
      let fmt = {
        prefix: "$",
        decimalSeparator: ".",
        groupSeparator: ",",
        groupSize: 3,
        secondaryGroupSize: 0,
        fractionGroupSeparator: " ",
        fractionGroupSize: 0,
        suffix: " ",
      };
      console.log("Format =>  Result:", BigNumber("123456789.123456789").dp(2).toFormat(fmt));
    </script>
  </body>
</html>
```

![](https://img2020.cnblogs.com/blog/464394/202104/464394-20210429181728383-2021171392.png)
