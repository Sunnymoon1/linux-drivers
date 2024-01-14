# linux-drivers
// SPDX-License-Identifier: GPL-2.0 /*

Self tests for device tree subsystem */
#define pr_fmt(fmt) "### dt-test ### " fmt

#include <linux/memblock.h> #include <linux/clk.h> #include <linux/dma-direct.h> /* to test phys_to_dma/dma_to_phys */ #include <linux/err.h> #include <linux/errno.h> #include <linux/hashtable.h> #include <linux/libfdt.h> #include <linux/of.h> #include <linux/of_address.h> #include <linux/of_fdt.h> #include <linux/of_irq.h> #include <linux/of_platform.h> #include <linux/list.h> #include <linux/mutex.h> #include <linux/slab.h> #include <linux/device.h> #include <linux/platform_device.h> #include <linux/kernel.h>

#include <linux/i2c.h> #include <linux/i2c-mux.h> #include <linux/gpio/driver.h>

#include <linux/bitops.h>

#include "of_private.h"

static struct unittest_results { int passed; int failed; } unittest_results;

#define unittest(result, fmt, ...) ({
bool failed = !(result);
if (failed) {
unittest_results.failed++;
pr_err("FAIL %s():%i " fmt, func, LINE, ##VA_ARGS);
} else {
unittest_results.passed++;
pr_info("pass %s():%i\n", func, LINE);
}
failed;
})

/*

Expected message may have a message level other than KERN_INFO.
Print the expected message only if the current loglevel will allow
the actual message to print.
Do not use EXPECT_BEGIN(), EXPECT_END(), EXPECT_NOT_BEGIN(), or
EXPECT_NOT_END() to report messages expected to be reported or not
reported by pr_debug(). */ #define EXPECT_BEGIN(level, fmt, ...)
printk(level pr_fmt("EXPECT \ : ") fmt, ##VA_ARGS)
#define EXPECT_END(level, fmt, ...)
printk(level pr_fmt("EXPECT / : ") fmt, ##VA_ARGS)
np = of_find_node_by_path("/testcase-data");
name = kasprintf(GFP_KERNEL, "%pOF", np);
unittest(np && !strcmp("/testcase-data", name),
	"find /testcase-data failed\n");
of_node_put(np);
kfree(name);

/* Test if trailing '/' works */
np = of_find_node_by_path("/testcase-data/");
unittest(!np, "trailing '/' on /testcase-data/ should fail\n");

np = of_find_node_by_path("/testcase-data/phandle-tests/consumer-a");
name = kasprintf(GFP_KERNEL, "%pOF", np);
unittest(np && !strcmp("/testcase-data/phandle-tests/consumer-a", name),
	"find /testcase-data/phandle-tests/consumer-a failed\n");
of_node_put(np);
kfree(name);

np = of_find_node_by_path("testcase-alias");
name = kasprintf(GFP_KERNEL, "%pOF", np);
unittest(np && !strcmp("/testcase-data", name),
	"find testcase-alias failed\n");
of_node_put(np);
kfree(name);

/* Test if trailing '/' works on aliases */
np = of_find_node_by_path("testcase-alias/");
unittest(!np, "trailing '/' on testcase-alias/ should fail\n");

np = of_find_node_by_path("testcase-alias/phandle-tests/consumer-a");
name = kasprintf(GFP_KERNEL, "%pOF", np);
unittest(np && !strcmp("/testcase-data/phandle-tests/consumer-a", name),
	"find testcase-alias/phandle-tests/consumer-a failed\n");
of_node_put(np);
kfree(name);

np = of_find_node_by_path("/testcase-data/missing-path");
unittest(!np, "non-existent path returned node %pOF\n", np);
of_node_put(np);

np = of_find_node_by_path("missing-alias");
unittest(!np, "non-existent alias returned node %pOF\n", np);
of_node_put(np);

np = of_find_node_by_path("testcase-alias/missing-path");
unittest(!np, "non-existent alias with relative path returned node %pOF\n", np);
of_node_put(np);

np = of_find_node_opts_by_path("/testcase-data:testoption", &options);
unittest(np && !strcmp("testoption", options),
	 "option path test failed\n");
of_node_put(np);

np = of_find_node_opts_by_path("/testcase-data:test/option", &options);
unittest(np && !strcmp("test/option", options),
	 "option path test, subcase #1 failed\n");
of_node_put(np);

np = of_find_node_opts_by_path("/testcase-data/testcase-device1:test/option", &options);
unittest(np && !strcmp("test/option", options),
	 "option path test, subcase #2 failed\n");
of_node_put(np);

np = of_find_node_opts_by_path("/testcase-data:testoption", NULL);
unittest(np, "NULL option path test failed\n");
of_node_put(np);

np = of_find_node_opts_by_path("testcase-alias:testaliasoption",
			       &options);
unittest(np && !strcmp("testaliasoption", options),
	 "option alias path test failed\n");
of_node_put(np);

np = of_find_node_opts_by_path("testcase-alias:test/alias/option",
			       &options);
unittest(np && !strcmp("test/alias/option", options),
	 "option alias path test, subcase #1 failed\n");
of_node_put(np);

np = of_find_node_opts_by_path("testcase-alias:testaliasoption", NULL);
unittest(np, "NULL option alias path test failed\n");
of_node_put(np);

options = "testoption";
np = of_find_node_opts_by_path("testcase-alias", &options);
unittest(np && !options, "option clearing test failed\n");
of_node_put(np);

options = "testoption";
np = of_find_node_opts_by_path("/", &options);
unittest(np && !options, "option clearing root node test failed\n");
of_node_put(np);

#define EXPECT_NOT_BEGIN(level, fmt, ...)
printk(level pr_fmt("EXPECT_NOT \ : ") fmt, ##VA_ARGS)

#define EXPECT_NOT_END(level, fmt, ...)
printk(level pr_fmt("EXPECT_NOT / : ") fmt, ##VA_ARGS)

static void __init of_unittest_find_node_by_name(void) { struct device_node *np; const char *options, *name;
np = of_find_node_by_path("/testcase-data");
if (!np) {
	pr_err("missing testcase data\n");
	return;
}

/* Array of 4 properties for the purpose of testing */
prop = kcalloc(4, sizeof(*prop), GFP_KERNEL);
if (!prop) {
	unittest(0, "kzalloc() failed\n");
	return;
}

/* Add a new property - should pass*/
prop->name = "new-property";
prop->value = "new-property-data";
prop->length = strlen(prop->value) + 1;
unittest(of_add_property(np, prop) == 0, "Adding a new property failed\n");

/* Try to add an existing property - should fail */
prop++;
prop->name = "new-property";
prop->value = "new-property-data-should-fail";
prop->length = strlen(prop->value) + 1;
unittest(of_add_property(np, prop) != 0,
	 "Adding an existing property should have failed\n");

/* Try to modify an existing property - should pass */
prop->value = "modify-property-data-should-pass";
prop->length = strlen(prop->value) + 1;
unittest(of_update_property(np, prop) == 0,
	 "Updating an existing property should have passed\n");

/* Try to modify non-existent property - should pass*/
prop++;
prop->name = "modify-property";
prop->value = "modify-missing-property-data-should-pass";
prop->length = strlen(prop->value) + 1;
unittest(of_update_property(np, prop) == 0,
	 "Updating a missing property should have passed\n");

/* Remove property - should pass */
unittest(of_remove_property(np, prop) == 0,
	 "Removing a property should have passed\n");

/* Adding very large property - should pass */
prop++;
prop->name = "large-property-PAGE_SIZEx8";
prop->length = PAGE_SIZE * 8;
prop->value = kzalloc(prop->length, GFP_KERNEL);
unittest(prop->value != NULL, "Unable to allocate large buffer\n");
if (prop->value)
	unittest(of_add_property(np, prop) == 0,
		 "Adding a large property should have passed\n");
