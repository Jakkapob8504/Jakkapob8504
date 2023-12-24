- 👋 Hi, I’m @Jakkapob8504
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
Jakkapob8504/Jakkapob8504 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
---># YANG data

This directory contains artifacts which deal with representing YANG-modeled data in ways similar
to how Java models XML data. These includes:
* an object model representation similar to W3C Document Object Model as found in **org.w3c.dom**
* corresponding stream similar Streaming APIs for XML as found in **java.xml.stream**
* an implementation of a YANG datastore, which has a number of convenient features:
  * transactions with multi-version concurrency control
  * optional enforcement of simple YANG structural constraints:
    * **mandatory**
    * **min-elements**/**max-elements**
/*
 * Copyright (c) 2015 Cisco Systems, Inc. and others.  All rights reserved.
 *
 * This program and the accompanying materials are made available under the
 * terms of the Eclipse Public License v1.0 which accompanies this distribution,
 * and is available at http://www.eclipse.org/legal/epl-v10.html
 */
package org.opendaylight.yangtools.yang.data.tree.impl;

import static com.google.common.base.Preconditions.checkState;
import static java.util.Objects.requireNonNull;

import java.util.Collection;
import java.util.Iterator;
import org.opendaylight.yangtools.yang.data.tree.impl.node.Version;

abstract class AbstractReadyIterator {
    final Iterator<ModifiedNode> children;
    final ModifiedNode node;
    final ModificationApplyOperation op;

    private AbstractReadyIterator(final ModifiedNode node, final Iterator<ModifiedNode> children,
            final ModificationApplyOperation operation) {
        this.children = requireNonNull(children);
        this.node = requireNonNull(node);
        this.op = requireNonNull(operation);
    }

    static AbstractReadyIterator create(final ModifiedNode root, final ModificationApplyOperation operation) {
        return new RootReadyIterator(root, root.getChildren().iterator(), operation);
    }

    final AbstractReadyIterator process(final Version version) {
        // Walk all child nodes and remove any children which have not
        // been modified. If a child has children, we need to iterate
        // through it via re-entering this method on the child iterator.
        while (children.hasNext()) {
            final ModifiedNode child = children.next();
            final ModificationApplyOperation childOp = op.childByArg(child.getIdentifier());
            checkState(childOp != null, "Schema for child %s is not present.", child.getIdentifier());
            final Collection<ModifiedNode> grandChildren = child.getChildren();

            if (grandChildren.isEmpty()) {
                // The child is empty, seal it
                child.seal(childOp, version);
                if (child.getOperation() == LogicalOperation.NONE) {
                    children.remove();
                }
            } else {
                return new NestedReadyIterator(this, child, grandChildren.iterator(), childOp);
            }
        }

        // We are done with this node, seal it.
        node.seal(op, version);

        // Remove from parent if we have one and this is a no-op
        if (node.getOperation() == LogicalOperation.NONE) {
            removeFromParent();
        }

        // Sub-iteration complete, return back to parent
        return getParent();
    }

    abstract AbstractReadyIterator getParent();

    abstract void removeFromParent();

    private static final class NestedReadyIterator extends AbstractReadyIterator {
        private final AbstractReadyIterator parent;

        private NestedReadyIterator(final AbstractReadyIterator parent, final ModifiedNode node,
                final Iterator<ModifiedNode> children, final ModificationApplyOperation operation) {
            super(node, children, operation);
            this.parent = requireNonNull(parent);
        }

        @Override
        AbstractReadyIterator getParent() {
            return parent;
        }

        @Override
        void removeFromParent() {
            parent.children.remove();
        }
    }

    private static final class RootReadyIterator extends AbstractReadyIterator {
        private RootReadyIterator(final ModifiedNode node, final Iterator<ModifiedNode> children,
                final ModificationApplyOperation operation) {
            super(node, children, operation);
        }

        @Override
        AbstractReadyIterator getParent() {
            return null;
        }

        @Override
        void removeFromParent() {
            // No-op, since root node cannot be removed
        }
    }
}
