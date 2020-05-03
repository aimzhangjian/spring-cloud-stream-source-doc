类工具
================================================
ClassUtils是一个非常大的工具类，提供了很多缓存数据和初始化内容

获取默认类加载器
------------------------------------------------

该方法用于获取默认的类加载器；方法功能很明确，平时也用的比较多，可能平时我们要获取类加载器，就一个class.getClassLoader()就完了，我们来看看spring是如何考虑的，spring按照获取当前线程上下文类加载器—>获取当前类类加载器—>获取系统启动类加载器的顺序来获取::

	/**
	 * Return the default ClassLoader to use: typically the thread context
	 * ClassLoader, if available; the ClassLoader that loaded the ClassUtils
	 * class will be used as fallback.
	 * <p>Call this method if you intend to use the thread context ClassLoader
	 * in a scenario where you clearly prefer a non-null ClassLoader reference:
	 * for example, for class path resource loading (but not necessarily for
	 * {@code Class.forName}, which accepts a {@code null} ClassLoader
	 * reference as well).
	 * @return the default ClassLoader (only {@code null} if even the system
	 * ClassLoader isn't accessible)
	 * @see Thread#getContextClassLoader()
	 * @see ClassLoader#getSystemClassLoader()
	 */
	@Nullable
	public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
		    //获取当前线程的context class loader
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
			// Cannot access thread context ClassLoader - falling back...
		}
		if (cl == null) {
			// No thread context class loader -> use class loader of this class.
			// 如果没有context loader，使用当前类的类加载器
			cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				// getClassLoader() returning null indicates the bootstrap ClassLoader
				try {
				    // 如果当前类加载器无法获取，获得bootstrap ClassLoader
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
				}
			}
		}
		return cl;
	}

获取默认类加载器
------------------------------------------------

这个方法较为简单，使用传入的classloader替换线程的classloader；使用场景，比如一个线程的classloader和spring的classloader不一致的时候，就可以使用这个方法替换::

    /**
     * Override the thread context ClassLoader with the environment's bean ClassLoader
     * if necessary, i.e. if the bean ClassLoader is not equivalent to the thread
     * context ClassLoader already.
     * @param classLoaderToUse the actual ClassLoader to use for the thread context
     * @return the original thread context ClassLoader, or {@code null} if not overridden
     */
    @Nullable
    public static ClassLoader overrideThreadContextClassLoader(@Nullable ClassLoader classLoaderToUse) {
        Thread currentThread = Thread.currentThread();
        ClassLoader threadContextClassLoader = currentThread.getContextClassLoader();
        if (classLoaderToUse != null && !classLoaderToUse.equals(threadContextClassLoader)) {
            //设置当前线程上下文类加载器为指定类加载器
            currentThread.setContextClassLoader(classLoaderToUse);
            return threadContextClassLoader;
        }
        else {
            return null;
        }
    }


通过类名从指定的类加载器获取类
------------------------------------------------

Class.forName的一个增强版本；通过指定的classloader加载对应的类；除了能正常加载普通的类型，还能加载简单类型，数组，或者内部类::

    /**
     * Replacement for {@code Class.forName()} that also returns Class instances
     * for primitives (e.g. "int") and array class names (e.g. "String[]").
     * Furthermore, it is also capable of resolving inner class names in Java source
     * style (e.g. "java.lang.Thread.State" instead of "java.lang.Thread$State").
     * @param name the name of the Class
     * @param classLoader the class loader to use
     * (may be {@code null}, which indicates the default class loader)
     * @return a class instance for the supplied name
     * @throws ClassNotFoundException if the class was not found
     * @throws LinkageError if the class file could not be loaded
     * @see Class#forName(String, boolean, ClassLoader)
     */
    public static Class<?> forName(String name, @Nullable ClassLoader classLoader)
            throws ClassNotFoundException, LinkageError {

        Assert.notNull(name, "Name must not be null");

        Class<?> clazz = resolvePrimitiveClassName(name);
        if (clazz == null) {
            clazz = commonClassCache.get(name);
        }
        if (clazz != null) {
            return clazz;
        }

        // "java.lang.String[]" style arrays
        if (name.endsWith(ARRAY_SUFFIX)) {
            String elementClassName = name.substring(0, name.length() - ARRAY_SUFFIX.length());
            Class<?> elementClass = forName(elementClassName, classLoader);
            return Array.newInstance(elementClass, 0).getClass();
        }

        // "[Ljava.lang.String;" style arrays
        if (name.startsWith(NON_PRIMITIVE_ARRAY_PREFIX) && name.endsWith(";")) {
            String elementName = name.substring(NON_PRIMITIVE_ARRAY_PREFIX.length(), name.length() - 1);
            Class<?> elementClass = forName(elementName, classLoader);
            return Array.newInstance(elementClass, 0).getClass();
        }

        // "[[I" or "[[Ljava.lang.String;" style arrays
        if (name.startsWith(INTERNAL_ARRAY_PREFIX)) {
            String elementName = name.substring(INTERNAL_ARRAY_PREFIX.length());
            Class<?> elementClass = forName(elementName, classLoader);
            return Array.newInstance(elementClass, 0).getClass();
        }

        ClassLoader clToUse = classLoader;
        if (clToUse == null) {
            clToUse = getDefaultClassLoader();
        }
        try {
            return (clToUse != null ? clToUse.loadClass(name) : Class.forName(name));
        }
        catch (ClassNotFoundException ex) {
            int lastDotIndex = name.lastIndexOf(PACKAGE_SEPARATOR);
            if (lastDotIndex != -1) {
                String innerClassName =
                        name.substring(0, lastDotIndex) + INNER_CLASS_SEPARATOR + name.substring(lastDotIndex + 1);
                try {
                    return (clToUse != null ? clToUse.loadClass(innerClassName) : Class.forName(innerClassName));
                }
                catch (ClassNotFoundException ex2) {
                    // Swallow - let original exception get through
                }
            }
            throw ex;
        }
    }

通过类名从指定的类加载器获取类
------------------------------------------------

和forName方法相同，内部就是直接调用的forName方法，只是抛出的异常不一样而已::

    /**
     * Resolve the given class name into a Class instance. Supports
     * primitives (like "int") and array class names (like "String[]").
     * <p>This is effectively equivalent to the {@code forName}
     * method with the same arguments, with the only difference being
     * the exceptions thrown in case of class loading failure.
     * @param className the name of the Class
     * @param classLoader the class loader to use
     * (may be {@code null}, which indicates the default class loader)
     * @return a class instance for the supplied name
     * @throws IllegalArgumentException if the class name was not resolvable
     * (that is, the class could not be found or the class file could not be loaded)
     * @see #forName(String, ClassLoader)
     */
    public static Class<?> resolveClassName(String className, @Nullable ClassLoader classLoader)
            throws IllegalArgumentException {

        try {
            return forName(className, classLoader);
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalArgumentException("Could not find class [" + className + "]", ex);
        }
        catch (LinkageError err) {
            throw new IllegalArgumentException("Unresolvable class definition for class [" + className + "]", err);
        }
    }

获取简单类型的类
------------------------------------------------

获取简单类型的类；这个方法是用于处理forName方法中简单类型的调用方法::

    /**
     * Resolve the given class name as primitive class, if appropriate,
     * according to the JVM's naming rules for primitive classes.
     * <p>Also supports the JVM's internal class names for primitive arrays.
     * Does <i>not</i> support the "[]" suffix notation for primitive arrays;
     * this is only supported by {@link #forName(String, ClassLoader)}.
     * @param name the name of the potentially primitive class
     * @return the primitive class, or {@code null} if the name does not denote
     * a primitive class or primitive array class
     */
    @Nullable
    public static Class<?> resolvePrimitiveClassName(@Nullable String name) {
        Class<?> result = null;
        // Most class names will be quite long, considering that they
        // SHOULD sit in a package, so a length check is worthwhile.
        if (name != null && name.length() <= 8) {
            // Could be a primitive - likely.
            result = primitiveTypeNameMap.get(name);
        }
        return result;
    }

primitiveTypeNameMap::

    /**
     * Map with primitive type name as key and corresponding primitive
     * type as value, for example: "int" -> "int.class".
     */
    private static final Map<String, Class<?>> primitiveTypeNameMap = new HashMap<>(32);

在ClassUtils类中，声明了这样一个静态的map；而他的初始化方法可以看到，其实就是把简单类型对应的名字和类型直接放到primitiveTypeNameMap中即可::

    static {
        primitiveWrapperTypeMap.put(Boolean.class, boolean.class);
        primitiveWrapperTypeMap.put(Byte.class, byte.class);
        primitiveWrapperTypeMap.put(Character.class, char.class);
        primitiveWrapperTypeMap.put(Double.class, double.class);
        primitiveWrapperTypeMap.put(Float.class, float.class);
        primitiveWrapperTypeMap.put(Integer.class, int.class);
        primitiveWrapperTypeMap.put(Long.class, long.class);
        primitiveWrapperTypeMap.put(Short.class, short.class);

        // Map entry iteration is less expensive to initialize than forEach with lambdas
        for (Map.Entry<Class<?>, Class<?>> entry : primitiveWrapperTypeMap.entrySet()) {
            primitiveTypeToWrapperMap.put(entry.getValue(), entry.getKey());
            registerCommonClasses(entry.getKey());
        }

        Set<Class<?>> primitiveTypes = new HashSet<>(32);
        primitiveTypes.addAll(primitiveWrapperTypeMap.values());
        Collections.addAll(primitiveTypes, boolean[].class, byte[].class, char[].class,
                double[].class, float[].class, int[].class, long[].class, short[].class);
        primitiveTypes.add(void.class);
        for (Class<?> primitiveType : primitiveTypes) {
            primitiveTypeNameMap.put(primitiveType.getName(), primitiveType);
        }

        registerCommonClasses(Boolean[].class, Byte[].class, Character[].class, Double[].class,
                Float[].class, Integer[].class, Long[].class, Short[].class);
        registerCommonClasses(Number.class, Number[].class, String.class, String[].class,
                Class.class, Class[].class, Object.class, Object[].class);
        registerCommonClasses(Throwable.class, Exception.class, RuntimeException.class,
                Error.class, StackTraceElement.class, StackTraceElement[].class);
        registerCommonClasses(Enum.class, Iterable.class, Iterator.class, Enumeration.class,
                Collection.class, List.class, Set.class, Map.class, Map.Entry.class, Optional.class);

        Class<?>[] javaLanguageInterfaceArray = {Serializable.class, Externalizable.class,
                Closeable.class, AutoCloseable.class, Cloneable.class, Comparable.class};
        registerCommonClasses(javaLanguageInterfaceArray);
        javaLanguageInterfaces = new HashSet<>(Arrays.asList(javaLanguageInterfaceArray));
    }