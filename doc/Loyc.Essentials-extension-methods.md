---
title: Loyc.Essentials Extension Methods
layout: default
---
Document unfinished.

Loyc.Essentials supplements the standard LINQ stuff (<tt>System.Linq.Enumerable</tt>) with a bunch more extension methods. At this point it's all stuff that I found useful personally, but I do take suggestions for additional extension methods.

First of all, here are the "core" extension methods mentioned in the <a href="http://loyc-etc.blogspot.ca/2014/02/using-loycessentials-collection.html">previous article</a>:

    public static partial class LCInterfaces // Loyc.Collections namespace
    {
      public static T TryGet<T>(this IListSource<T> list,
                                int index, T defaultValue);
      public static bool TryGet<T>(this IListSource<T> list, 
                                   int index, ref T value);

      public static int IndexOf<(this IReadOnlyList<T> list, T item)
      public static void CopyTo<T>(this IReadOnlyList<T> c, T[] array, int arrayIndex)

      // Stack/Queue stuff
      public static T Pop<T>(this IPop<T> c);
      public static T Peek<T>(this IPop<T> c);
      public static bool TryPop<T>(this IPop<T> c, out T value);
      public static bool TryPeek<T>(this IPop<T> c, out T value);
      public static T TryPop<T>(this IPop<T> c);
      public static T TryPop<T>(this IPop<T> c, T defaultValue);
      public static T TryPeek<T>(this IPop<T> c);
      public static T TryPeek<T>(this IPop<T> c, T defaultValue);

      // Dequeue stuff
      public static T PopFirst<T>(this IDeque<T> c)
      public static T PopLast<T>(this IDeque<T> c)
      public static T PeekFirst<T>(this IDeque<T> c)
      public static T PeekLast<T>(this IDeque<T> c)
      public static bool TryPopFirst<T>(this IDeque<T> c, out T value)
      public static bool TryPopLast<T>(this IDeque<T> c, out T value)
      public static bool TryPeekFirst<T>(this IDeque<T> c, out T value)
      public static bool TryPeekLast<T>(this IDeque<T> c, out T value)
      public static T TryPopFirst<T>(this IDeque<T> c)
      public static T TryPopFirst<T>(this IDeque<T> c, T defaultValue)
      public static T TryPopLast<T>(this IDeque<T> c)
      public static T TryPopLast<T>(this IDeque<T> c, T defaultValue)
      public static T TryPeekFirst<T>(this IDeque<T> c)
      public static T TryPeekFirst<T>(this IDeque<T> c, T defaultValue)
      public static T TryPeekLast<T>(this IDeque<T> c)
      public static T TryPeekLast<T>(this IDeque<T> c, T defaultValue)
    }

### 

### Extension methods for Dictionary/IDictionary (Loyc.Collections.DictionaryExt)

	public static V TryGetValue<K, V>(this IDictionary<K, V> dict, K key, V defaultValue)
	public static V TryGetValue<K, V>(this Dictionary<K, V> dict, K key, V defaultValue) {
		V value;
		if (key == null || !dict.TryGetValue(key, out value))
			return defaultValue;
		return value;
	}

An alternate version TryGetValue that returns `defaultValue` if the key was not found in the dictionary, and that does not throw if the key is null.		

	public static bool TryGetValueSafe<K, V>(this IDictionary<K, V> dict, K key, out V value)
	{
		if (key != null)
			return dict.TryGetValue(key, out value);
		value = default(V);
		return false;
	}


Same as `IDictionary.TryGetValue()` except that this method does not throw an exception when `key==null`.


