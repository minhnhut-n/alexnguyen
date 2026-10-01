==================
Sorting Algorithms
==================

Tài liệu về các thuật toán sắp xếp: Selection Sort, Insertion Sort, Quick Sort và
Merge Sort, kèm mã nguồn minh họa bằng ngôn ngữ C và phân tích độ phức tạp.

.. meta::
   :description: Tài liệu chi tiết về các thuật toán sắp xếp (Sorting Algorithms): Selection Sort, Insertion Sort, Quick Sort và Merge Sort kèm mã nguồn C minh họa.
   :keywords: Sorting, Sorting Algorithms, Selection Sort, Insertion Sort, Quick Sort, Merge Sort, Big-O, Complexity, C

.. contents:: **Mục lục**
   :depth: 2
   :local:

---

.. list-table:: **Bảng so sánh độ phức tạp các thuật toán sắp xếp**
   :widths: 28 24 24 24 20
   :header-rows: 1
   :align: center

   * - Thuật toán
     - Tốt nhất
     - Trung bình
     - Xấu nhất
     - Bộ nhớ
   * - **Selection Sort**
     - :math:`O(n^2)`
     - :math:`O(n^2)`
     - :math:`O(n^2)`
     - :math:`O(1)`
   * - **Insertion Sort**
     - :math:`O(n)`
     - :math:`O(n^2)`
     - :math:`O(n^2)`
     - :math:`O(1)`
   * - **Quick Sort**
     - :math:`O(n \log n)`
     - :math:`O(n \log n)`
     - :math:`O(n^2)`
     - :math:`O(\log n)`
   * - **Merge Sort**
     - :math:`O(n \log n)`
     - :math:`O(n \log n)`
     - :math:`O(n \log n)`
     - :math:`O(n)`

---

1. Selection Sort (``insertS.c``)
---------------------------------

.. code-block:: c

   /*
    thuật toán sort có bigo là O(n^2)
    min là giá trị i, tăng lên sau mỗi vòng loop
    tìm giá trị j = i+1 --> max size array, có giá trị nhỏ hơn
    và hoán đổi vị trí, cho đến khi không còn phần tử.
   */

   #include <stdio.h>

   int input[] = {1, 2, 9, 8, 4, 3, 1};

   int main() {
       int size = sizeof(input) / sizeof(input[0]);
       for (int i = 0; i < size; i++) {
           int min = i;
           for (int j = i + 1; j < size; j++) {
               if (input[min] > input[j]) {
                   int temp = input[min];
                   input[min] = input[j];
                   input[j] = temp;
               }
           }
       }

       for (int i = 0; i < size; i++)
           printf("%d ", input[i]);
       return 0;
   }

---

2. Insertion Sort (``selectionS.c``)
------------------------------------

.. code-block:: c

   /**
    * selection sort
    * for each value (index), check with it previous value
    * bubble sort back with i-1,
    * until meet condition.
    */
   #include <stdio.h>

   int main() {
       int a[] = {64, 34, 25, 12, 22, 11, 90, 5};
       int n = sizeof(a) / sizeof(a[0]);

       for (int i = 0; i < n; i++) {
           while (i - 1 >= 0 && a[i - 1] > a[i]) {
               int temp = a[i];
               a[i] = a[i - 1];
               a[i - 1] = temp;
               i--;
           }
       }

       for (int i = 0; i < n; i++)
           printf("%d ", a[i]);
       return 0;
   }

---

3. Quick Sort (``quickS.c``)
----------------------------

.. code-block:: c

   /**
    * quick sort
    * idea from choosing 1 pivot value (in the middle)
    * left is ideally less than pivot
    * right is ideally more than pivot
    *
    * first sort from left to right to find the middle position which is fine for this condition
    * then swap to pivot index
    *
    * after exit recursive, result will be showed
    */
   #include <stdio.h>

   void swap(int *arr, int i1, int i2) {
       int tmp = arr[i1];
       arr[i1] = arr[i2];
       arr[i2] = tmp;
   }

   void sort(int *arr, int left, int right) {
       if (left >= right) return;

       int pivotIndex = right;
       int mark = left - 1;
       for (int i = left; i < right; i++) {
           if (arr[i] < arr[pivotIndex]) {
               mark++;
               swap(arr, mark, i);
           }
       }
       pivotIndex = mark + 1;
       // swap to pivot
       swap(arr, right, mark + 1);

       sort(arr, left, pivotIndex - 1);
       sort(arr, pivotIndex + 1, right);
   }

   int main() {
       int a[] = {64, 34, 25, 12, 22, 11, 90, 5};
       int n = sizeof(a) / sizeof(a[0]);

       sort(a, 0, n - 1);

       for (int i = 0; i < n; i++)
           printf("%d ", a[i]);
       return 0;
   }

---

4. Merge Sort (``mergeS.c``)
----------------------------

.. code-block:: c

   /**
    * merge sort
    * in place sort/ migrate
    */
   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>

   void theMerge(int* arr, int i1, int i2, int j1, int j2) {
       int start = i1;
       int total = j2 - i1 + 1;
       int* temp = (int*) calloc(total, sizeof(int));
       if (temp == NULL) {
           return;
       }

       int idx = 0;
       while (i1 <= i2 && j1 <= j2) {
           if (arr[i1] <= arr[j1]) {
               temp[idx] = arr[i1];
               i1++;
           }
           else {
               temp[idx] = arr[j1];
               j1++;
           }
           idx++;
       }

       while (i1 <= i2) {
           //assign first then count
           temp[idx++] = arr[i1++];
       }
       while (j1 <= j2) {
           temp[idx++] = arr[j1++];
       }

       memcpy(&arr[start], &temp[0], total * sizeof(int));
       free(temp);
   }

   /**
    * behave sort 2 sides of pivot index (left and right side)
    * pivot is choice as the middle of array,
    * 1. recursive call sort in left and right
    * 2. merge left and right then return this recursive (the left and last recursive first
    * then trace back to the begining)
    */

   void sort(int *arr, int left, int right) {
       int pivot = (left + right) / 2;

       if (left >= right) {
           return;
           // the signal for nothing to divide any more.
           // larger is not expected to happen.
       }

       sort(arr, left, pivot);
       //array list left
       sort(arr, pivot + 1, right);
       //array list right
       theMerge(arr, left, pivot, pivot + 1, right);
   }

   int main() {
       int a[] = {64, 34, 25, 12, 22, 11, 90, 5};
       int n = sizeof(a) / sizeof(a[0]);
       sort(a, 0, n - 1);

       for (int i = 0; i < n; i++)
           printf("%d ", a[i]);
       return 0;
   }

---

5. Merge Sort - GPT Optimized (``mergeS_gpt.c``)
------------------------------------------------

.. code-block:: c

   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>

   void theMerge(int *arr, int *temp, int left, int mid, int right)
   {
       int i = left;
       int j = mid + 1;
       int k = left;

       while (i <= mid && j <= right) {
           if (arr[i] <= arr[j]) {
               temp[k++] = arr[i++];
           } else {
               temp[k++] = arr[j++];
           }
       }

       while (i <= mid) {
           temp[k++] = arr[i++];
       }
       while (j <= right) {
           temp[k++] = arr[j++];
       }

       memcpy(&arr[left],
              &temp[left],
              (right - left + 1) * sizeof(int));
   }

   void mergeSort(int *arr, int *temp, int left, int right)
   {
       if (left >= right) {
           return;
       }

       int mid = left + (right - left) / 2;
       mergeSort(arr, temp, left, mid);
       mergeSort(arr, temp, mid + 1, right);
       // Already sorted across the boundary
       if (arr[mid] <= arr[mid + 1]) {
           return;
       }
       theMerge(arr, temp, left, mid, right);
   }

   void sort(int *arr, int n)
   {
       int *temp = malloc(n * sizeof(int));
       if (temp == NULL) {
           return;
       }
       mergeSort(arr, temp, 0, n - 1);
       free(temp);
   }

   int main(void)
   {
       int a[] = {64, 34, 25, 12, 22, 11, 90, 5};
       int n = sizeof(a) / sizeof(a[0]);

       sort(a, n);

       for (int i = 0; i < n; i++) {
           printf("%d ", a[i]);
       }

       return 0;
   }
