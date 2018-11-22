#!/usr/bin/env python3

"""
Store the given url in an org-mode file
Note: This script must be called from qutebrowser
"""

import PyOrgMode as pyorg
import os.path
import subprocess
import shlex
import sys
import argparse
import re
import time

ORG_FILE = os.path.expanduser("~/read-later.org")
HEADING_PATH = "Read Later"
HEADING_SEPERATOR = '/'
TIME_FORMAT = "Captured: <%Y-%m-%d %a>"
DMENU = "dmenu -i -l 15"

argparser = argparse.ArgumentParser(description=__doc__)
argparser.add_argument("mode", nargs='?', choices=["write", "read", "rm"])
argparser.add_argument("--re", "-r", nargs='?', default="")

def qute_command(command):
    """send commands to qutebrowser"""
    with open(os.environ['QUTE_FIFO'], 'w') as fifo:
        fifo.write(command + '\n')
        fifo.flush

def dmenu(items, invocation):
    """run dmenu"""
    command = shlex.split(invocation)
    process = subprocess.run(command,
                             input='\n'.join(items).encode("UTF-8"),
                             stdout=subprocess.PIPE)
    return process.stdout.decode("UTF-8").strip()

def new_node(heading, level):
    """Return a new orgnode"""
    node = pyorg.OrgNode.Element()
    node.heading = heading
    node.level = level
    return node

def resolve_heading(org, headings):
    """
    return the node for the specified subheading from org
    nested subheadings can be specified using `/' e.g. heading/subheading
    create any headings that don't exist
    """
    def subheading(org, heading):
        """return the node for heading immediately below org"""
        for c in org.content:
            if c.heading == heading:
                return c
        return None

    node = org
    for h in headings.split(HEADING_SEPERATOR):
        if subheading(node, h):
            node = subheading(node, h)
        else:
            nnode = new_node(h, node.level + 1)
            node.append_clean(nnode)
            node = nnode
    return node

def node_select_dmenu(org, rexp):
    """Return a node under org selected with dmenu"""
    # Nodes -> text -> node
    # we prepend the dmenu entry with an index back into the array
    # arbitrarily large limit
    n = (n for n in range(0, 10**10))
    if org.content:
        items = ["{}. {} {}".format(next(n),
                                    e.content[0].strip(),
                                    e.heading.strip()) for e in org.content]

        # ideally we should be matching against the header too
        if rexp:
            items = [i for i in items if re.search(rexp, i)]

        selection = dmenu(items, DMENU).split(' ')[0]
        if selection:
            return org.content[int(selection.split('.')[0])]
    return None

def main(args):
    """
    Store given url in an org-mode file
    or interactively retrieve using dmenu
    """
    if not args.mode or not os.getenv("QUTE_URL"):
        argparser.print_help()
        return 1

    org = pyorg.OrgDataStructure()
    title = os.getenv("QUTE_TITLE")
    url = os.getenv("QUTE_URL")
    content = os.getenv("QUTE_SELECTED_TEXT")

    try:
        org.load_from_file(ORG_FILE);
    except FileNotFoundError:
        # If the file doesn't exist we begin with an empty org structure
        pass

    node = resolve_heading(org.root, HEADING_PATH);

    if args.mode == "write":
        newnode = new_node(title, node.level + 1)
        newnode.append_clean(url + '\n')
        newnode.append_clean(time.strftime(TIME_FORMAT) + '\n')
        if content:
            newnode.append_clean(content + '\n')
        # we append a final newline to ensure each element is space separated
        # this may be personal preference
        newnode.append_clean('\n')
        node.append_clean(newnode)
        org.save_to_file(ORG_FILE)
        qute_command("message-info \"captured " + title + "\"")

    elif args.mode == "read":
        selection = node_select_dmenu(node, args.re)
        if selection:
            qute_command("open -t " + selection.content[0].strip())

    elif args.mode == "rm":
        selection = node_select_dmenu(node, args.re)
        if selection:
            node.content.remove(selection)
            org.save_to_file(ORG_FILE)
            qute_command("message-info \"removed: " + selection.heading + "\"")

    return 0

if __name__ == '__main__':
    sys.exit(main(argparser.parse_args()))
