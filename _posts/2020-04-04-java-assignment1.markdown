---
layout: post
title: "[Java] 한 줄로 10의 자리와 1의 자리 숫자 비교하기"
category: dev
tags: dev-java
comments: true
---

# Problem
>두 자리 정수(10~99)를 입력받아 10의 자리와 1의 자리가 같다면 메세지를 출력하시오.  
>-*(명품 JAVA Programming p.110 No.2)*

# Solve
일반적으로 생각할 수 있는 해결 방법은 Integer형 변수를 선언하여 ```%``` 연산자와 ```/```연산자를 사용하여 비교하는 방법일 것이다.
```java
import java.util.*;

public class Example02 {
    public static void main(String[] args) {
        int val = new Scanner(System.in).nextInt();
        if(val / 10 == val % 10)
            System.out.println("Yes! 10의 자리와 1의 자리가 같습니다.");
    }
}
```
하지만, 과제를 수행하던 중 ***한 줄로 짤 수는 없을까?*** 라는 쓸데없는 생각이 들어서 짜 보았다.
```java
import java.util.*;
public class Example02 {
    public static void main(String[] args) {
        if(new HashSet<>(new ArrayList<>(Arrays.asList(new Scanner(System.in).next().split("")))).size() == 1) System.out.println("Yes! 10의 자리와 1의 자리가 같습니다.");
    }
}
```

그렇다. 한 줄이라는 의의 외에는 아무런 메리트도 찾아볼 수 없는 코드다.
```Scanner.next()```에서 받은 ```String``` 데이터를 ```String Array```로, 다시 ```Java.util.Arrays$ArrayList```로, 다시 ```java.util.ArrayList```로, 마지막으로 ```HashSet```으로 중복 데이터를 제거하여 그 결과가 1개일 경우 출력하도록 한 코드이다.

결론적으로, 해당 기능을 한줄로 아주 기괴하게 작성하는 데 성공했다는 점에서 매우 만족스럽다.