默认参数 / Default arguments
Kotlin 的允许提供变量的默认值，构造时可以选择不传参直接使用默认值。

data class Person(var age: Int = 28, var name: String, var nickName: String? = null)
 
fun main() {
    val p1 = Person(name = "YJY")
    val p2 = Person(25, "YJY")
    val p3 = Person(25, name = "YJY", nickName = "Max")
}
命名参数 / Named Arguments
在 Kotlin 我们可以选择不传递已有默认值的参数，也可以使用命名参数来按随意顺序来传递参数。

fun reformat(
    str: String,
    normalizeCase: Boolean = true,
    upperCaseFirstLetter: Boolean = true,
    divideByCamelHumps: Boolean = false,
    wordSeparator: Char = ' ',
) {/*...*/}
 
fun main() {
    //normalizeCase、upperCaseFirstLetter 默认为 true
    //命名参数后，不需要按方法签名的参数顺序传参
    reformat("message", wordSeparator = '|', divideByCamelHumps = true)
}
可变数量的参数 / Varargs
传递不定数量的参数，示例是一个计算所有 Int 参数总和的方法

fun cal(vararg numbers: Int) {
    var sum = 0
    numbers.forEach {
        sum += it
    }
    println("result=[$sum]")
}
 
fun main() {
    cal(1, 2, 3, 4, 5)
}
单行表达式函数 / Single-expression functions
顾名思义，方法只有一行代码。

fun isAdult(person: Person): Boolean = person.age >= 18
局部函数 / Local functions
方法套方法，局部函数可以访问外部函数(闭包)的局部变量，太骚气了

fun isUserAllowToAccessWebsite(user: User): Boolean {
 
    fun isAdult(): Boolean {
        return user.age >= 18
    }
 
    //用户未成年
    if (!isAdult()) {
        return false
    }
 
    //...其他判断
    return true
}
