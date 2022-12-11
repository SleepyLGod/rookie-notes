# 😱 bignumer.js中常见运算



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
      // 判断类型是否是BigNumber
      console.log("判断类型是否是BigNumber => BigNumber.isBigNumber(n) Result:", BigNumber.isBigNumber(n));
      console.log("判断类型是否是BigNumber => BigNumber.isBigNumber(x) Result:", BigNumber.isBigNumber(x));
      // 取最大值
      console.log("取最大值 => BigNumber.max(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.max(12, 2, 3, 34, 55).toNumber());
      console.log("取最大值 => BigNumber.maximum(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.maximum(12, 2, 3, 34, 55).toNumber());
      // 取最小值
      console.log("取最小值 => BigNumber.min(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.min(12, 2, 3, 34, 55).toNumber());
      console.log("取最小值 => BigNumber.minimum(12, 2, 3, 34, 55).toNumber() Result:", BigNumber.minimum(12, 2, 3, 34, 55).toNumber());
      // 求和
      console.log("求和 => BigNumber.sum(4e9, x, '1').toNumber() Result:", BigNumber.sum(4e9, x, "1").toNumber());
      // 2位小于1随机数
      console.log("2位小于1随机数 => BigNumber.random(2).toNumber() Result:", BigNumber.random(2).toNumber());
      // 绝对值
      console.log("绝对值 => BigNumber('-1.21').abs().toNumber() Result:", BigNumber("-1.21").abs().toNumber());
      // 比较大小
      // 1    如果这个大数字的价值大于n
      // -1    如果这个大数字的价值低于n
      // 0    如果这个大数字和有相同的价值n
      // null    如果这个大数字的价值或是nNaN
      console.log("比较大小 => BigNumber(111111).comparedTo(2222) Result:", BigNumber(111111).comparedTo(2222));
      // 四舍五入
      // 三位小数
      console.log("四舍五入 三位小数 => BigNumber(12.32162).decimalPlaces(3).toNumber() Result:", BigNumber(12.32162).decimalPlaces(3).toNumber());
      console.log("四舍五入 三位小数 => BigNumber(12.32162).dp(3).toNumber() Result:", BigNumber(12.32162).dp(3).toNumber());

      // 四舍五入 整数
      console.log("四舍五入 整数 => BigNumber(123.756).integerValue().toNumber() Result:", BigNumber(123.756).integerValue().toNumber());

      // 等于
      console.log("等于 => BigNumber(123).isEqualTo(123.00000000000000001) Result:", BigNumber(123).isEqualTo(123.00000000000000001));
      console.log("等于 => BigNumber(123).isEqualTo(123.100000000000000001) Result:", BigNumber(123).isEqualTo(123.100000000000000001));
      console.log("等于 => BigNumber(123).eq(123.00000000000000001) Result:", BigNumber(123).eq(123.00000000000000001));
      console.log("等于 => BigNumber(123).eq(123.100000000000000001) Result:", BigNumber(123).eq(123.100000000000000001));

      // 大于
      console.log("大于 => BigNumber(0.3).minus(0.2).isGreaterThan(0.1) Result:", BigNumber(0.3).minus(0.2).isGreaterThan(0.1));
      console.log("大于 => BigNumber(0.3).minus(0.2).gt(0.1) Result:", BigNumber(0.3).minus(0.2).gt(0.1));

      // 大于等于
      console.log("大于等于 => BigNumber(0.3).minus(0.2).isGreaterThanOrEqualTo(0.1) Result:", BigNumber(0.3).minus(0.2).isGreaterThanOrEqualTo(0.1));
      console.log("大于等于 => BigNumber(0.3).minus(0.2).gte(0.1) Result:", BigNumber(0.3).minus(0.2).gte(0.1));

      // 小于
      console.log("小于 => BigNumber(0.3).minus(0.2).isLessThan(0.1) Result:", BigNumber(0.3).minus(0.2).isLessThan(0.1));
      console.log("小于 => BigNumber(0.3).minus(0.2).lt(0.1) Result:", BigNumber(0.3).minus(0.2).lt(0.1));

      // 小于等于
      console.log("小于等于 => BigNumber(0.3).minus(0.2).isLessThanOrEqualTo(0.1) Result:", BigNumber(0.3).minus(0.2).isLessThanOrEqualTo(0.1));
      console.log("小于等于 => BigNumber(0.3).minus(0.2).lte(0.1) Result:", BigNumber(0.3).minus(0.2).lte(0.1));

      // 加
      console.log("加 => BigNumber(0.2).plus(0.1).toNumber() Result:", BigNumber(0.2).plus(0.1).toNumber());
      // 减
      console.log("减 => BigNumber(0.3).minus(0.1).toNumber() Result:", BigNumber(0.3).minus(0.1).toNumber());
      // 乘
      console.log("乘 => BigNumber(0.3).multipliedBy(0.2).toNumber() Result:", BigNumber(0.3).multipliedBy(0.2).toNumber());
      console.log("乘 => BigNumber(0.3).times(0.2).toNumber() Result:", BigNumber(0.3).times(0.2).toNumber());
      // 除
      console.log("除 => BigNumber(12).dividedBy(7).toNumber() Result:", BigNumber(12).dividedBy(7).toNumber());
      console.log("除 => BigNumber(12).div(7).toNumber() Result:", BigNumber(12).div(7).toNumber());
      // 除 取整数部分
      console.log("除 取整数部分 => BigNumber(12).dividedToIntegerBy(7).toNumber() Result:", BigNumber(12).dividedToIntegerBy(7).toNumber());
      console.log("除 取整数部分 => BigNumber(12).idiv(7).toNumber() Result:", BigNumber(12).idiv(7).toNumber());

      // 取余 10 % 3
      console.log("取余 => BigNumber(10).modulo(3).toNumber() Result:", BigNumber(10).modulo(3).toNumber());
      console.log("取余 => BigNumber(10).mod(3).toNumber() Result:", BigNumber(10).mod(3).toNumber());

      // 取反数 -1.3 * -1
      console.log("取反数 -1.3 * -1 => BigNumber(-1.3).negated().toNumber() Result:", BigNumber(-1.3).negated().toNumber());

      // 指数
      console.log("指数 2^10 => BigNumber(2).exponentiatedBy(10).toNumber() Result:", BigNumber(2).exponentiatedBy(10).toNumber());
      console.log("指数 2^10 => BigNumber(2).pow(10).toNumber() Result:", BigNumber(2).pow(10).toNumber());

      // 是否是有限的数字
      console.log("是否是有限的数字 =>  Result:", BigNumber(123).isFinite());
      console.log("是否是有限的数字 =>  Result:", BigNumber(Infinity).isFinite());

      // 是否是一个整数
      console.log("是否是一个整数 =>  Result:", BigNumber(0.3).isInteger());
      console.log("是否是一个整数 =>  Result:", BigNumber(3).isInteger());

      // 是否是NaN
      console.log("是否是NaN =>  Result:", BigNumber(NaN).isNaN());
      console.log("是否是NaN =>  Result:", BigNumber(Infinity).isNaN());

      // 是否是负数
      console.log("是否是负数 =>  Result:", BigNumber(-0).isNegative());
      console.log("是否是负数 =>  Result:", BigNumber(0).isNegative());

      // 格式化
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
      console.log("格式化 =>  Result:", BigNumber("123456789.123456789").dp(2).toFormat(fmt));
    </script>
  </body>
</html>
```

![](https://s2.loli.net/2022/07/24/UMR2nT81swWHCoS.png)
