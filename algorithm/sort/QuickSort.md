##排序算法
参考博客: https://www.cnblogs.com/onepixel/p/7674659.html
## 快速排序
快速排序参考博客: https://www.jianshu.com/p/7631d95fdb0b

````java
public class QuickSortTest {


    public static int partition(int[] arrays, int first, int j) {
        int last = j - 1;
        for (int i = first + 1; i <= last; ) {
            if (arrays[first] <= arrays[last]) {
                last--;
                continue;
            }
            if (arrays[first] > arrays[i]) {
                i++;
                continue;
            }
            swap(arrays, i, last);

        }
        swap(arrays, first, last);
        return last;
    }

    private static void quickSort(int[] arrays, int first, int last) {
        if (first >= last   ) {
            return;
        }
        int index = partition(arrays, first, last);
        if (index > first) {
            quickSort(arrays, first, index);
        }
        if (index < last) {
            quickSort(arrays, index + 1, last);
        }
    }

    static void swap(int[] arrays, int i, int j) {
        int tmp = arrays[j];
        arrays[j] = arrays[i];
        arrays[i] = tmp;
    }

    public static void main(String[] args) {
        int[] arrays = {5, 4, 7, 8, 2, 7, 8, 5, 6, 3};
        quickSort(arrays, 0, arrays.length);
        System.out.println(Arrays.toString(arrays));
    }
}
````