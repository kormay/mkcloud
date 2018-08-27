```python
class Solution:
    def twoSum(self, nums, target):
        """
        :type nums: List[int]
        :type target: int
        :rtype: List[int]
        """
        temp_dict = {}
        for i, item in enumerate(nums):
            if target - item in temp_dict:
                return [i, temp_dict[target - item]]
            temp_dict[item] = i

        return []

```
